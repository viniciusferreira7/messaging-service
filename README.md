# Messaging Service

> 🚧 **Work In Progress** — This project is in early development. Only the RabbitMQ infrastructure is in place; the service implementation and its stack are not defined yet.

Asynchronous messaging backbone for the Marketplace microservices architecture. Carries domain events between services and handles delivery concerns — routing, retries with backoff, and dead-lettering of messages that cannot be processed.

Part of the **marketplace-ms** study project, alongside the [api-gateway](https://github.com/viniciusferreira7/api-gateway), which is the synchronous (HTTP) entry point of the same architecture.

## Scope

- **Event broker between services** — publishes and consumes domain events (e.g. `order.created`, `payment.approved`) so `users`, `products`, `checkouts`, and `payments` stay decoupled from each other.
- **Dead-letter and retry** — failed messages are retried with backoff and, once the attempt budget is exhausted, parked in a dead-letter queue for inspection and reprocessing.

## Architecture

The gateway handles synchronous request/response traffic; this service handles everything that happens after the response is returned.

```
Client
  │  HTTP
  ▼
API Gateway  ──────────────► users · products · checkouts · payments
                                     │              ▲
                                     │ publish      │ consume
                                     ▼              │
                            ┌────────────────────────────┐
                            │         RabbitMQ           │
                            │                            │
                            │  exchange → queues         │
                            │     │                      │
                            │     ├── retry (backoff)    │
                            │     └── dead-letter queue  │
                            └────────────────────────────┘
```

> The diagram describes the intended design. Nothing beyond the broker container exists in this repository yet.

## Requirements

- [Docker](https://www.docker.com/) and Docker Compose

## Running the broker

```bash
# start RabbitMQ
docker compose up -d

# follow the logs
docker compose logs -f rabbitmq

# stop
docker compose down
```

The image is `rabbitmq:4.3.3-management-alpine`, so the management UI ships with it.

### Environment variables

Credentials are read from the environment, with defaults for local development:

| Variable                | Description       | Default |
|-------------------------|-------------------|---------|
| `RABBITMQ_DEFAULT_USER` | RabbitMQ username | `admin` |
| `RABBITMQ_DEFAULT_PASS` | RabbitMQ password | `admin` |

> The defaults are for local use only. Set real credentials through the environment before running this anywhere else.

## Roadmap

- [ ] Define the service stack and scaffold the project
- [ ] Expose broker ports and persist data with a named volume
- [ ] Model exchanges, queues, and routing keys for the domain events
- [ ] Publisher and consumer with connection/channel lifecycle handling
- [ ] Retry with exponential backoff and jitter
- [ ] Dead-letter queue and a reprocessing path
- [ ] Consumer idempotency
- [ ] Health checks and observability
- [ ] Tests

## Related repositories

| Repository | Description |
|------------|-------------|
| [api-gateway](https://github.com/viniciusferreira7/api-gateway) | Central HTTP entry point for the Marketplace microservices |
