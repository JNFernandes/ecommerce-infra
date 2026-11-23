# ecommerce-infra

[![CI](https://github.com/JNFernandes/ecommerce-infra/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/JNFernandes/ecommerce-infra/actions/workflows/ci.yml)
[![Security](https://github.com/JNFernandes/ecommerce-infra/actions/workflows/security.yml/badge.svg?branch=main)](https://github.com/JNFernandes/ecommerce-infra/actions/workflows/security.yml)

Local runtime for the ecommerce platform: a set of microservices that place an order, charge it
and reserve its stock, coordinated only through Kafka events (a choreographed saga). This
repository holds the shared infrastructure and a `docker-compose.yml` that can also build and run
the services from their sibling repositories.

| Service | Repository | Stack | Role |
|---|---|---|---|
| Orders | `../ecommerce-orders` | NestJS, TypeORM | REST API to place, cancel and confirm orders; GraphQL read model; joins the payment and stock outcomes |
| Payments | `../ecommerce-payments` | .NET worker, EF Core | Charges each placed order (fails above 10 000.00), refunds cancelled orders |
| Inventory | `../ecommerce-inventory` | Spring Boot, Flyway | Reserves stock for each placed order, releases it when the order is cancelled; read-only products API |
| Notifications | `../ecommerce-notifications` | Python, FastAPI, MongoDB | Records a customer notification for each order event (sending is simulated); read-only HTTP API |
| Identity | `../ecommerce-identity` | NestJS, TypeORM | Users, credentials and sessions; issues the RS256 JWTs (JWKS at `/.well-known/jwks.json`); publishes `identity.user-registered` |
| AI | `../ecommerce-ai` | Python, FastAPI, PostgreSQL | Planned: MCP tools over the platform APIs, triage of failed orders (`ai.order-triage-completed`), a conversational assistant. Bootstrap today: health and readiness only |

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

   orders.order-placed / -confirmed / -cancelled, payments.payment-refunded
                                │  (from Kafka)
                       ┌────────▼────────┐  writes   ┌───────────────────────┐
                       │  Notifications  ├──────────►│ mongo (notifications) │
                       └────────▲────────┘           └───────────────────────┘
                                │
                REST :8000  GET /notifications (read-only)
```

An order starts `PENDING`. It becomes `CONFIRMED` when the payment is processed **and** the stock
is reserved, and `CANCELLED` as soon as either fails. A cancellation makes Payments refund a
processed payment and Inventory release a reserved stock. Notifications follows the saga without
taking part in it: it records one notification per order event and publishes nothing. AI
(planned) will also follow it from the side, explaining failed orders, and calls the APIs through
the gateway only with the caller's own token. Topics, payloads, the saga and the Notifications read
API are specified in [docs/events.md](docs/events.md).

## Prerequisites

- Docker with Docker Compose v2 (`docker compose`).
- For the `apps` profile: the service repositories cloned next to this one:

  ```
  ecommerce-project/
  ├── ecommerce-infra/        (this repository)
  ├── ecommerce-orders/
  ├── ecommerce-payments/
  ├── ecommerce-inventory/
  ├── ecommerce-notifications/
  ├── ecommerce-identity/
  └── ecommerce-ai/
  ```

- For Identity: Node.js 20+ on the host, once, to generate its signing key (see [First run](#first-run)).
- Free host ports: 3000, 3001, 5432, 5433, 5434, 5435, 5436, 8000, 8002, 8080 (only with `tools`), 8081, 9092, 27017.

## First run

Identity signs its tokens with an RS256 private key that is never built into the image nor
committed. Generate it once, in the Identity repository (`keys/` is git-ignored there):

```bash
cd ../ecommerce-identity
npm ci
npm run keys:generate      # writes keys/private.pem and keys/public.pem
cd ../ecommerce-infra
docker compose --profile apps up -d --build
```

Compose mounts `../ecommerce-identity/keys` read-only at `/run/keys` in the container
(`JWT_PRIVATE_KEY_PATH=/run/keys/private.pem`). Without the key the `identity` container exits on
startup and keeps restarting.

To also create an administrator, set the admin variables in the host environment before `up` (see
[Identity admin](#identity-admin)).

## Run

Infrastructure only (Kafka, the five Postgres databases, MongoDB), for running the services from
your IDE against `localhost`:

```bash
docker compose up -d
```

Everything, with the services built from the sibling repositories:

```bash
docker compose --profile apps up -d --build
```

Add `--profile tools` to either command to also start Kafka UI on http://localhost:8080.

`docker compose --profile apps ps` should show `orders`, `inventory`, `notifications`, `identity` and `ai` as `healthy`,
`payments` and `gateway` as `Up` (no healthcheck). Add `-a` to also see `kafka-init` and `orders-seed`,
which should both be `Exited (0)`.

What happens on start:

1. Kafka, Postgres and MongoDB start and report healthy.
2. `kafka-init` creates the topics (idempotent) and exits.
3. The services start once their database and Kafka are healthy and `kafka-init` has finished.
   They restart on failure, so a service that loses Kafka at startup recovers.
4. Each service migrates its own schema on startup: Orders and Identity with TypeORM, Payments
   with EF Core, Inventory with Flyway. Notifications creates its MongoDB indexes (database
   `notifications`). AI has no schema yet; it starts without Kafka, PostgreSQL or an LLM key.
5. Demo data is loaded, idempotently (existing rows are never changed):
   - Orders: customer `3fa85f64-5717-4562-b3fc-2c963f66afa6`, by the one-shot `orders-seed`
     container (the `seed:customers` script), which runs once Orders is healthy and then exits.
   - Inventory: the sample stock below (Spring profile `local`, `db/local/R__seed_local_stock.sql`).
   - Identity: one `ADMIN`, only when none exists and `ADMIN_EMAIL` / `ADMIN_PASSWORD` are set
     (see [Identity admin](#identity-admin)).

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
| API gateway (`apps` profile; see [API gateway](#api-gateway)) | `localhost:8090/api/...` | | | |
| Orders (REST, GraphQL at `/graphql`, Swagger at `/api`) | via gateway `/api/orders`, `/api/graphql` | `orders:3000` | | |
| Inventory (products API, Swagger UI at `/swagger-ui.html`, health at `/actuator/health`) | via gateway `/api/inventory/products` | `inventory:8081` | | |
| Notifications (read API, docs at `/docs`, health at `/health/ready`) | via gateway `/api/notifications` | `notifications:8000` | | |
| Identity (auth API, Swagger UI at `/docs`, health at `/health`, JWKS at `/.well-known/jwks.json`) | via gateway `/api/identity/...` | `identity:3001` | | |
| AI (docs at `/docs`, health at `/health`, readiness at `/ready`) | no gateway route yet | `ai:8002` | | |
| Payments | not published | | | |
| Kafka | `localhost:9092` | `kafka:29092` | | |
| Kafka UI (`tools` profile) | `localhost:8080` | | | |
| Postgres Orders | `localhost:5432` | `postgres-orders:5432` | `orders` | `orders_user` / `orders_pass` |
| Postgres Inventory | `localhost:5433` | `postgres-inventory:5432` | `inventory` | `inventory_user` / `inventory_pass` |
| Postgres Payments | `localhost:5434` | `postgres-payments:5432` | `payments` | `payments_user` / `payments_pass` |
| Postgres Identity | `localhost:5435` | `postgres-identity:5432` | `identity` | `identity_user` / `identity_pass` |
| Postgres AI | `localhost:5436` | `postgres-ai:5432` | `ecommerce_ai` | `ai_user` / `ai_pass` |
| MongoDB | `localhost:27017` | `mongo:27017` | `orders_read` (Orders read model), `notifications` (Notifications) | `mongo_user` / `mongo_pass`, auth database `admin` |

With `docker-compose.dev.yml` the five HTTP services are also published on the internal port
(`localhost:3000`, `3001`, `8000`, `8002`, `8081`). These credentials are for local development only.
Notifications consumes in the Kafka group
`notifications-service`.

### Identity admin

At startup Identity creates one `ADMIN` when no admin exists yet and both `ADMIN_EMAIL` and
`ADMIN_PASSWORD` are set. Compose passes these three variables through from the host environment,
with no default password; when they are unset no admin is created (Identity logs a warning) and
everything else works. Existing users are never changed or promoted.

| Variable | Required | Notes |
|---|---|---|
| `ADMIN_EMAIL` | for the seed | Same rules as registration (trimmed, lowercased). |
| `ADMIN_PASSWORD` | for the seed | 10–128 characters, not equal to the email. Never logged. Not needed after the first start. |
| `ADMIN_NAME` | no | Display name, default `Administrator`. |

```bash
ADMIN_EMAIL=admin@example.com ADMIN_PASSWORD='choose-a-long-password' \
  docker compose --profile apps up -d --build
```

```powershell
$env:ADMIN_EMAIL = 'admin@example.com'
$env:ADMIN_PASSWORD = 'choose-a-long-password'
docker compose --profile apps up -d --build
Remove-Item Env:ADMIN_PASSWORD
```

The seed runs only while no admin exists, so after a reset (`down -v`) set the variables again.

Compose also sets `JWT_ISSUER=http://localhost:3001` and `JWT_AUDIENCE=ecommerce`, the values
other services must expect when they verify Identity's tokens.

## HTTP APIs

Four services have an HTTP API for clients such as a browser frontend, reached through the
[API gateway](#api-gateway) at `http://localhost:8090/api/...` (the table lists the paths inside
each service; the gateway prefix is in brackets). Identity issues RS256
access tokens; Orders and Notifications verify them locally against Identity's public keys
(`IDENTITY_JWKS_URL=http://identity:3001/.well-known/jwks.json`, fetched on the first
authenticated request and cached, never at startup). Inventory's catalog is public. Services never
call each other's APIs: Kafka events remain the only service-to-service communication (see
[docs/events.md](docs/events.md#http-apis)).

| Service | Endpoints | Interactive docs |
|---|---|---|
| Orders (`/api/orders`, `/api/graphql`) | `POST /orders`, `POST /orders/{orderId}/cancel`, `POST /orders/{orderId}/confirm`; GraphQL `POST /graphql` with the queries `order(orderId)` and `orders(status, customerId, limit, offset)` | Swagger UI: `gateway-debug` profile, 127.0.0.1:9000/api |
| Inventory (`/api/inventory`) | `GET /products?limit=&offset=`, `GET /products/{id}` (404 if unknown); `GET /actuator/health` internal only | Swagger UI: `gateway-debug` profile, 127.0.0.1:9081/swagger-ui/index.html |
| Notifications (`/api/notifications`) | `GET /notifications?customerId=&orderId=&type=&limit=&offset=`, `GET /notifications/{id}` (404 if unknown); `GET /health/live`, `GET /health/ready` internal only | Docs: `gateway-debug` profile, 127.0.0.1:9080/docs |
| Identity (`/api/identity`) | `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`, `GET /users/me`, the admin `/users` and `/audit` endpoints (full list in the Identity README); `GET /.well-known/jwks.json`, `GET /health` internal only | Swagger UI: `gateway-debug` profile, 127.0.0.1:9001/docs |

The issuer and audience are defined once in `docker-compose.yml` (`x-jwt`, defaults
`http://localhost:3001` and `ecommerce`, overridable with `JWT_ISSUER` / `JWT_AUDIENCE` in the host
environment) and shared by Identity, Orders and Notifications, so they cannot drift.
`JWT_CLOCK_TOLERANCE_SECONDS` (default `5`) sets the allowed clock skew in Orders and
Notifications. Both start without Identity (`depends_on` is `service_started` only).

### Access rules

| Service | Anonymous | `CUSTOMER` | `ADMIN` |
|---|---|---|---|
| Identity | `/auth/register`, `/auth/login`, `/auth/refresh`, `/auth/logout`, JWKS, `/health`, `/docs` | `GET /users/me`, `POST /users/me/password` | as customer, plus `/users`, `/users/{id}`, `/users/{id}/status`, `/users/{id}/roles`, `/audit` |
| Orders | Swagger only; every API call is 401 (GraphQL: `UNAUTHENTICATED` error) | `POST /orders` for themselves (no `customerId` in the body); cancel own `PENDING` orders; GraphQL `order` / `orders` return only their own (another customer's order is `null`, cancelling it 404) | every order in GraphQL (`customerId` filter allowed); cancel and confirm any order; cannot place orders (403) |
| Notifications | `/health/live`, `/health/ready`, `/docs`; `/notifications` is 401 | only their own notifications (another customer's are invisible: empty list / 404) | every notification, any `customerId` filter |
| Inventory | `GET /products`, `GET /products/{id}`, health (no authentication) | same | same |
| Payments | no HTTP API | — | — |

### CORS

The gateway answers CORS for every API: `GATEWAY_CORS_ORIGINS` (comma-separated exact origins,
default `http://localhost:5173`). Set it in the host environment before `up`:

```bash
GATEWAY_CORS_ORIGINS=http://localhost:5173,http://localhost:4200 docker compose --profile apps up -d
```

```powershell
$env:GATEWAY_CORS_ORIGINS = 'http://localhost:5173,http://localhost:4200'
docker compose --profile apps up -d
```

Behind the gateway the services' own `CORS_ALLOWED_ORIGINS` is empty, so they add no CORS headers
of their own. With `docker-compose.dev.yml` (direct access) it is `http://localhost:5173` again
(override with `CORS_ALLOWED_ORIGINS`). Any other origin gets no `Access-Control-Allow-Origin`, so
the browser blocks the call. Check it with:

```bash
curl -s -D - -o /dev/null -X OPTIONS localhost:8090/api/orders -H 'Origin: http://localhost:5173'   -H 'Access-Control-Request-Method: POST' | grep -i access-control
```

### Kafka topics

Created by `kafka-init` with 3 partitions and replication factor 1 (auto-creation is off):
`orders.order-placed`, `orders.order-confirmed`, `orders.order-cancelled`,
`payments.payment-processed`, `payments.payment-failed`, `payments.payment-refunded`,
`inventory.stock-reserved`, `inventory.stock-rejected`, `inventory.stock-released`,
`identity.user-registered`, `ai.order-triage-completed` (planned publisher: AI).

## API gateway

With `--profile apps`, a Traefik gateway is the only published entry to the application APIs:
**http://localhost:8090/api/...**. Orders, Identity, Inventory, Notifications and AI publish no host
port (the databases, Kafka and MongoDB keep theirs for the infra-only workflow). Configuration
only, in [`gateway/`](gateway) (mounted read-only): `traefik.yml` (static) and
`dynamic/routes.yml` (routes, middlewares). Traefik uses the file provider only: the Docker socket
is not mounted, and the dashboard and API are off.

### Routes

| Gateway path | Service | Forwarded as | Notes |
|---|---|---|---|
| `/api/identity/auth/...` | `identity:3001` | `/auth/...` | register, login, refresh, logout |
| `/api/identity/users[/...]` | `identity:3001` | `/users[/...]` | `/users/me`, `/users/me/password`, admin user endpoints |
| `/api/identity/audit[/...]` | `identity:3001` | `/audit[/...]` | admin |
| `/api/orders[/...]` | `orders:3000` | `/orders[/...]` | place, cancel, confirm |
| `/api/graphql` | `orders:3000` | `/graphql` | read model |
| `/api/inventory/products[/...]` | `inventory:8081` | `/products[/...]` | public catalog |
| `/api/notifications[/...]` | `notifications:8000` | `/notifications[/...]` | |

AI has no route yet: it is added with its first API (the assistant or the MCP endpoint). Its
SSE streams will need a longer write timeout than the entrypoint's 30 s.

Everything else answers `404` at the gateway: JWKS (`/.well-known/jwks.json`, which Orders and
Notifications fetch over the internal network), Swagger UIs and OpenAPI documents, `/health*`,
`/actuator/*`, the Traefik dashboard.

### Middlewares

Every route goes through, in order:

| Middleware | Setting |
|---|---|
| Rate limit (global) | per client IP: 50 req/s average, burst 100 → `429` |
| Rate limit (auth) | only `/api/identity/auth/login`, `/auth/register`, `/auth/refresh`, `/users/me/password`: 10 per minute per IP, burst 5 → `429` |
| CORS | origins from `GATEWAY_CORS_ORIGINS` (comma-separated, exact match; default `http://localhost:5173`); methods `GET POST PUT PATCH DELETE OPTIONS`; headers `Authorization`, `Content-Type`; no credentials; preflight answered by the gateway |
| Security headers | `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, `Permissions-Policy: camera=(), microphone=(), geolocation=()`, `Content-Security-Policy: default-src 'none'; frame-ancestors 'none'`; `Server` and `X-Powered-By` removed. HSTS is prepared but commented out (TLS only) |
| Body limit | request bodies over 1 MB → `413` (buffered) |
| Compression | gzip/brotli when the client accepts it |

Timeouts: 30 s to read a request and to write a response, 60 s idle keep-alive (entrypoint);
5 s to connect and 30 s for response headers towards a service. Access logs go to stdout as JSON
(`docker compose logs gateway`) with `Authorization` and `Cookie` written as `REDACTED`.

### Direct access for debugging

`docker-compose.dev.yml` publishes the services' ports again (`3000`, `3001`, `8000`, `8002`, `8081`) and
sets their own `CORS_ALLOWED_ORIGINS` to `http://localhost:5173`. The gateway keeps working
alongside:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml --profile apps up -d
```

To go back, run the plain `docker compose --profile apps up -d`. The `gateway-debug` profile
starts a second Traefik bound to `127.0.0.1` only, with the dashboard and the Swagger UIs:

```bash
docker compose --profile apps --profile gateway-debug up -d
```

| URL | What |
|---|---|
| http://127.0.0.1:8091/dashboard/ | Traefik dashboard (of the debug instance) |
| http://127.0.0.1:9000/api | Orders Swagger UI |
| http://127.0.0.1:9001/docs | Identity Swagger UI |
| http://127.0.0.1:9081/swagger-ui/index.html | Inventory Swagger UI |
| http://127.0.0.1:9080/docs | Notifications docs |
| http://127.0.0.1:9002/docs | AI docs |

These serve the documentation only; "Try it out" calls are not routed there. Use the gateway URLs
instead, or the dev override.

### Security notes

- **The gateway** is the single entry point. It limits request rates and body sizes, applies
  timeouts, answers CORS, adds the security headers, hides `Server` / `X-Powered-By`, and exposes
  only the API routes above. It discards `X-Forwarded-*` headers sent by clients and sets its own.
- **The services still** verify every access token (signature against Identity's JWKS, issuer,
  audience, expiry), check roles and ownership, validate input, and apply their own CORS policy
  (empty behind the gateway, so they add no CORS headers) and Identity's own login and
  registration throttling. The gateway is defense in depth, not the only control: a request that
  reaches a service by another path is still checked.
- **JWT validation stays in the services.** Each service decides who may do what with its own data
  (a customer's own orders, admin-only routes), which needs the token's claims at the service
  anyway; validating only at the edge would make every service trust whatever reaches it on the
  internal network. The gateway stays configuration-only and stateless.
- **Identity throttling behind the gateway.** Identity limits by the TCP client address, which is
  now the gateway's, so its per-IP buckets are shared by all clients (its per-email login limit
  still applies). The gateway's per-IP auth limit restores per-client limiting at the edge. Making
  Identity trust the gateway's `X-Forwarded-For` would need a change in Identity.
- **In production** TLS is terminated at the gateway (an HTTPS entrypoint with certificates),
  HTTP redirects to HTTPS, and HSTS is enabled by uncommenting `stsSeconds` in
  `gateway/dynamic/routes.yml`. The debug profile and the dev override are never deployed.

## End-to-end demo

Every order belongs to the customer in the access token, so the flow is: register, log in, place
orders with the token, read them back. The seeded customer `3fa85f64-…` has no Identity account and
cannot be used for new orders. The saga takes a second or two, so a query right after the `POST`
may still show `PENDING` or fewer notifications; query again.

### 1. Keys and stack (from empty volumes)

```powershell
cd ..\ecommerce-identity; npm ci; npm run keys:generate; cd ..\ecommerce-infra
docker compose --profile apps down -v
$env:ADMIN_EMAIL = 'admin@example.com'; $env:ADMIN_PASSWORD = 'Admin-Passw0rd!'; $env:ADMIN_NAME = 'Admin'
docker compose --profile apps up -d --build
Remove-Item Env:ADMIN_PASSWORD
docker compose --profile apps ps      # orders, inventory, notifications, identity: healthy
```

```bash
(cd ../ecommerce-identity && npm ci && npm run keys:generate)
docker compose --profile apps down -v
ADMIN_EMAIL=admin@example.com ADMIN_PASSWORD='Admin-Passw0rd!' ADMIN_NAME=Admin \
  docker compose --profile apps up -d --build
```

### 2. Customer: register, log in, order, read

PowerShell (`curl` is an alias of `Invoke-WebRequest` in Windows PowerShell, so use
`Invoke-RestMethod`):

```powershell
$cred = @{ email = 'alice@example.com'; password = 'a-long-password-1' }
Invoke-RestMethod -Method Post http://localhost:8090/api/identity/auth/register -ContentType 'application/json' `
  -Body (@{ email = $cred.email; name = 'Alice'; password = $cred.password } | ConvertTo-Json)
$tok = (Invoke-RestMethod -Method Post http://localhost:8090/api/identity/auth/login -ContentType 'application/json' -Body ($cred | ConvertTo-Json)).accessToken
$h = @{ Authorization = "Bearer $tok" }

# Place an order (no customerId: it comes from the token)
$body = @{ items = @(@{ productId = 'SKU-003'; quantity = 2; unitPrice = 19.99 }) } | ConvertTo-Json -Depth 5
$r = Invoke-RestMethod -Method Post http://localhost:8090/api/orders -Headers $h -ContentType 'application/json' -Body $body

# Read it with GraphQL
$q = @{ query = "{ order(orderId: `"$($r.orderId)`") { orderId status totalAmount cancellationReason items { productId quantity } } }" } | ConvertTo-Json
(Invoke-RestMethod -Method Post http://localhost:8090/api/graphql -Headers $h -ContentType 'application/json' -Body $q).data.order

# Read its notifications (newest first)
(Invoke-RestMethod "http://localhost:8090/api/notifications?orderId=$($r.orderId)" -Headers $h).items | Format-Table type, message
```

curl (bash, Git Bash, WSL):

```bash
curl -s -X POST localhost:8090/api/identity/auth/register -H 'content-type: application/json' \
  -d '{"email":"alice@example.com","name":"Alice","password":"a-long-password-1"}'
# → 201 {"id":"…","customerId":"…","email":"alice@example.com","name":"Alice","roles":["CUSTOMER"],…}
TOKEN=$(curl -s -X POST localhost:8090/api/identity/auth/login -H 'content-type: application/json' \
  -d '{"email":"alice@example.com","password":"a-long-password-1"}' | sed -E 's/.*"accessToken":"([^"]+)".*/\1/')

curl -s -X POST localhost:8090/api/orders -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"items":[{"productId":"SKU-003","quantity":2,"unitPrice":19.99}]}'
# → 201 {"orderId":"<id>"}

curl -s localhost:8090/api/graphql -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"query":"{ orders(limit: 10) { totalCount items { orderId status totalAmount cancellationReason } } }"}'

curl -s "localhost:8090/api/notifications?orderId=<id>" -H "Authorization: Bearer $TOKEN"
# → {"items":[{"id":"…","customerId":"…","orderId":"<id>","type":"ORDER_CONFIRMED","channel":"email",
#     "message":"Your order <id> was confirmed.","status":"SENT","createdAt":"…","sourceEventId":"…"},…],
#    "total":2,"limit":20,"offset":0}
```

Registration publishes `identity.user-registered`; Orders creates the customer from it. An order
placed in the first instant after registration can answer `404` (customer not known yet): retry.

### 3. Admin: every order, user and notification

```powershell
$adm = (Invoke-RestMethod -Method Post http://localhost:8090/api/identity/auth/login -ContentType 'application/json' `
  -Body (@{ email = 'admin@example.com'; password = 'Admin-Passw0rd!' } | ConvertTo-Json)).accessToken
$ha = @{ Authorization = "Bearer $adm" }
$q = @{ query = '{ orders(limit: 50) { totalCount items { orderId customerId status } } }' } | ConvertTo-Json
(Invoke-RestMethod -Method Post http://localhost:8090/api/graphql -Headers $ha -ContentType 'application/json' -Body $q).data.orders.items
(Invoke-RestMethod http://localhost:8090/api/identity/users -Headers $ha).items | Format-Table email, roles, status
(Invoke-RestMethod 'http://localhost:8090/api/notifications?limit=100' -Headers $ha).items | Format-Table customerId, type
```

```bash
ADMIN=$(curl -s -X POST localhost:8090/api/identity/auth/login -H 'content-type: application/json' \
  -d '{"email":"admin@example.com","password":"Admin-Passw0rd!"}' | sed -E 's/.*"accessToken":"([^"]+)".*/\1/')
curl -s localhost:8090/api/graphql -H "Authorization: Bearer $ADMIN" -H 'content-type: application/json' \
  -d '{"query":"{ orders(limit: 50) { totalCount items { orderId customerId status } } }"}'
curl -s localhost:8090/api/identity/users -H "Authorization: Bearer $ADMIN"
curl -s "localhost:8090/api/notifications?limit=100" -H "Authorization: Bearer $ADMIN"
```

A second customer (register and log in as `bob@example.com`) gets `totalCount: 0`, an empty
notification list, `order: null` for Alice's order id and `404` when cancelling it.

### Expected results

Place each order with the customer token, changing only `items`:

| Scenario | Items | `status`, `cancellationReason` | Side effects |
|---|---|---|---|
| Happy path | `SKU-003` × 2 at 19.99 | `CONFIRMED`, `null` | Payment `Processed`; `SKU-003` stock 100 → 98 |
| Payment failure | `SKU-001` × 1 at 15000 | `CANCELLED`, `payment failed: Amount 15000 USD exceeds the maximum allowed threshold of 10000.00 USD.` | Payment `Failed`; the reserved stock is released, `SKU-001` stays at 10 |
| Stock rejection | `SKU-005` × 1 at 9.99 (or `SKU-004` × 6) | `CANCELLED`, `stock rejected: insufficient stock` | Payment `Processed`, then `Refunded` |
| Unknown product | `SKU-999` × 1 at 9.99 | `CANCELLED`, `stock rejected: unknown product` | Payment `Processed`, then `Refunded` |

Notifications per order (`GET /notifications?orderId=<id>`), in recording order (the API returns
newest first):

| Scenario | Notifications (`type`: `message`) |
|---|---|
| Happy path | `ORDER_RECEIVED`: `Your order <id> for 39.98 was received.`<br>`ORDER_CONFIRMED`: `Your order <id> was confirmed.` |
| Payment failure | `ORDER_RECEIVED`: `Your order <id> for 15000.00 was received.`<br>`ORDER_CANCELLED`: `Your order <id> was cancelled: payment failed: Amount 15000 USD exceeds the maximum allowed threshold of 10000.00 USD.`<br>No refund: nothing was charged. |
| Stock rejection | `ORDER_RECEIVED`: `Your order <id> for 9.99 was received.`<br>`ORDER_CANCELLED`: `Your order <id> was cancelled: stock rejected: insufficient stock.`<br>`PAYMENT_REFUNDED`: `Your payment of 9.99 USD for order <id> was refunded.` |
| Unknown product | `ORDER_RECEIVED`: `Your order <id> for 9.99 was received.`<br>`ORDER_CANCELLED`: `Your order <id> was cancelled: stock rejected: unknown product.`<br>`PAYMENT_REFUNDED`: `Your payment of 9.99 USD for order <id> was refunded.` |

Every notification has `channel` `email` and `status` `SENT`; sending is simulated (stored and
logged only). Optional filters: `type` (`ORDER_RECEIVED`, `ORDER_CONFIRMED`, `ORDER_CANCELLED`,
`PAYMENT_REFUNDED`), `limit` (1–100, default 20), `offset`.

To watch the events, start with `--profile tools` and open Kafka UI → Topics. To check the
databases directly:

```bash
docker compose exec postgres-inventory psql -U inventory_user -d inventory -c "select * from stock_items order by 1"
docker compose exec postgres-payments psql -U payments_user -d payments -c 'select "OrderId","Status","Amount" from payments'
docker compose exec mongo mongosh -u mongo_user -p mongo_pass --authenticationDatabase admin notifications --eval 'db.notifications.find({}, {type: 1, orderId: 1, _id: 0})'
```

## CI and security

This repository holds configuration only (compose files, the Traefik gateway, docs): there is no
application code and no package manifest, so the services' quality gates (tests, linters,
dependency audits) run in their own repositories.

| Workflow | When | Jobs |
|---|---|---|
| [CI](.github/workflows/ci.yml) | pull requests, pushes to `main` | **Compose config**: every file and profile combination validates (the sibling repos are not needed). **Gateway smoke test**: starts the gateway alone and checks the edge rules (JWKS and dashboard not exposed, CORS allow/deny, `413` over 1 MB, security headers, `429` on the auth rate limit). **Workflow lint**: actionlint (with shellcheck). |
| [Security](.github/workflows/security.yml) | pull requests, pushes to `main`, Mondays 05:17 UTC | **Secrets**: gitleaks over the full history, fails on any finding. **CodeQL** for the workflows (language `actions`). **Image scan**: Trivy on every third-party image in the compose file (the list is read from `docker-compose.yml` on each run, so new services and Dependabot bumps are scanned without editing the workflow), SARIF in the Security tab. |
| [Dependabot](.github/dependabot.yml) | weekly | One grouped PR for the compose images, one for the GitHub Actions. |

Every action is pinned to a full commit SHA (version in a comment), and the tool images to a
digest; Dependabot keeps them current. Jobs run with `contents: read`, plus `security-events:
write` only where SARIF is uploaded.

Not applicable here, so not run: a dependency audit and OSV-Scanner (no lockfile or manifest to
scan) and a Dockerfile build (the images are built in the service repositories, which scan them).

**Image scan threshold.** Trivy reports HIGH and CRITICAL vulnerabilities that have a fix
(`ignore-unfixed`); unfixed and lower-severity ones are not reported. Only the gateway image
(`traefik`) fails the job: it is the only component exposed to clients, and its tag is ours to
bump. Postgres, MongoDB, Kafka and Kafka UI are local-development infrastructure. Their findings
mostly sit in upstream binaries (`gosu`, the MongoDB tools, the JDK and Python inside the
Confluent image) that only a new upstream image can fix, so they are uploaded to the Security tab
without failing the pipeline. Review them there and accept the Dependabot image bumps.

Run the same checks locally (Docker only):

```bash
docker compose --profile apps --profile tools --profile gateway-debug config -q
docker compose -f docker-compose.yml -f docker-compose.dev.yml --profile apps config -q
docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:1.7.12
docker run --rm -v "$PWD:/repo" ghcr.io/gitleaks/gitleaks:v8.30.1 git /repo --redact -v
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image \
  --ignore-unfixed --severity HIGH,CRITICAL traefik:v3.7
```

The gateway smoke test is the script in `ci.yml` (job `gateway`), run against
`docker compose --profile apps up -d --no-deps gateway`.

**Handling a finding**

- **Secret (gitleaks)**: treat it as leaked. Revoke or rotate it first, then remove it from the
  code. Rewriting history does not undo the leak. Only for a proven false positive, add a
  `.gitleaksignore` entry with its fingerprint and the reason in the PR.
- **Image vulnerability (Trivy)**: bump the image tag in `docker-compose.yml` (or merge the
  Dependabot PR), re-run the scan, then verify the stack (`down -v`, `--profile apps up -d
  --build`, the end-to-end demo). If no fixed image exists yet, record why in the PR and dismiss
  the alert in the Security tab with a reason; never lower the threshold for everyone.
- **CodeQL (workflows)**: fix the workflow (typically untrusted input in a `run:` script, or
  excess permissions); dismiss only with a written reason.

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
- **`identity` keeps restarting**: `docker compose logs identity`. Usually the signing key is
  missing: run `npm run keys:generate` in `../ecommerce-identity` (see [First run](#first-run)).
  An invalid `ADMIN_EMAIL` / `ADMIN_PASSWORD` also stops startup, with a message naming the
  variable. On a Linux host the key is written with mode 0600, so it must be readable by uid 1000
  (the image's `node` user); Docker Desktop on Windows and macOS is not affected.
- **No notifications for an order**: run `docker compose exec notifications python -c "import urllib.request; print(urllib.request.urlopen('http://127.0.0.1:8000/health/ready').read().decode())"`. It answers 503 when
  MongoDB is unreachable, and `"kafka"` should be `CONNECTED` (`CONNECTING` or `DISCONNECTED`
  means the consumer is still retrying Kafka; the API keeps answering meanwhile). Then
  `docker compose logs -f notifications`: each recorded notification logs a line on
  `notifications.send`.
- **A message Notifications could not process**: invalid messages, and messages that still fail
  after the retries, are stored with the reason in the `inbound_dead_letters` collection:
  `docker compose exec mongo mongosh -u mongo_user -p mongo_pass --authenticationDatabase admin notifications --eval 'db.inbound_dead_letters.find()'`.
  A refund whose order placement was never received is dead-lettered with
  `errorType: "OrderContextNotFoundError"` and is not recorded later.
- **Browser call blocked by CORS**: the page's origin is not in `GATEWAY_CORS_ORIGINS` (default
  `http://localhost:5173`). Set it in the host environment and recreate the containers with
  `docker compose --profile apps up -d` (see [CORS](#cors)).
- **`port is already allocated` on 9092, 8090 or (with the dev override) 8000, 8081, 3001**: often another project's Kafka
  container, or a Notifications (`uvicorn --port 8000`), Inventory (Spring Boot, 8081) or Identity
  (`npm run start:dev`, 3001) dev server still running from the IDE. `docker ps` and
  `netstat -ano | findstr :8000` show the owner.
- **Identity keys missing**: `identity` keeps restarting and logs that the private key cannot be
  read. Run `npm run keys:generate` in `../ecommerce-identity`, then
  `docker compose --profile apps up -d identity`.
- **401 from Orders or Notifications**: no `Authorization: Bearer` header, a token older than 15
  minutes (log in again or use `/auth/refresh`), or a token from an Identity with another issuer or
  audience. The body never says which; check that `JWT_ISSUER` / `JWT_AUDIENCE` were not overridden
  for only one service (compose shares them through `x-jwt`).
- **403**: the token is valid but the role does not allow the call, e.g. an admin placing an order
  or a customer calling `/users`.
- **503 `authentication unavailable` / `SERVICE_UNAVAILABLE`**: Identity's keys have never been
  fetched since the service started, because Identity is down. Health endpoints stay 200 and events
  keep flowing. Start Identity (`docker compose start identity`); requests succeed again within
  about a minute (the key refetch is throttled). Keys already cached keep working while Identity is
  down.
- **`429` from the gateway**: more than 10 calls per minute to login, register, refresh or the
  password change from one IP (burst 5), or more than 50 req/s overall. Wait a minute.
- **`413`**: the request body is over 1 MB.
- **`404` from `localhost:8090`**: the path is not one of the [gateway routes](#routes) (Swagger,
  health and JWKS are not exposed; use the `gateway-debug` profile or `docker-compose.dev.yml`).
- **`connection refused` on 3000, 3001, 8000 or 8081**: expected; those ports are only published
  with `docker-compose.dev.yml`.
