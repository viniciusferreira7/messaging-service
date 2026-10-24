# Messaging Service

Asynchronous messaging backbone for the Marketplace microservices architecture: the shared RabbitMQ broker every service publishes to and consumes from.

Part of the **marketplace-ms** study project, alongside the [api-gateway](https://github.com/viniciusferreira7/api-gateway), which is the synchronous (HTTP) entry point of the same architecture.

## Scope

This repository holds the broker only. There is no application code: each service declares the exchanges and queues it needs when it connects, so the topology lives next to the code that depends on it.

- **Event broker between services** — the checkout service publishes payment orders and the payments service consumes them, so neither calls the other over HTTP.
- **Retry and dead-letter** — failed messages wait in a retry queue and come back; once the attempt budget is exhausted they are parked in a dead-letter queue, which payments-service exposes for inspection and reprocessing.

## Architecture

```
Client
  │  HTTP
  ▼
API Gateway (3333) ──► users · products · checkout · payments
                                             │          ▲
                                   publish   │          │ consume
                             payment.order   ▼          │
                            ┌─────────────────────────────────────────┐
                            │                RabbitMQ                 │
                            │                                         │
                            │  payments (topic) ──► payment_queue     │
                            │     ▲                    │ nack         │
                            │     │ after 30s          ▼              │
                            │  payment_queue.retry ◄── payments.retry.dlx
                            │                                         │
                            │  attempts exhausted ──► payments.dlx    │
                            │                          └► payment_queue.dlq (7 days)
                            └─────────────────────────────────────────┘
```

## Topology

Declared by payments-service on boot (the checkout service asserts the exchange too before publishing):

| Name | Kind | Purpose |
|------|------|---------|
| `payments` | topic exchange | Payment orders, routing key `payment.order` |
| `payment_queue` | queue | Consumed by payments-service; messages expire after 24 hours |
| `payments.retry.dlx` → `payment_queue.retry` | exchange + queue | Holds a failed message for `RABBITMQ_RETRY_DELAY_MS` (30 s) and sends it back to `payments` |
| `payments.dlx` → `payment_queue.dlq` | exchange + queue | Messages still failing after `RABBITMQ_RETRY_MAX_ATTEMPTS` (3) retries, 4 attempts in all; kept for 7 days |

## Requirements

- [Docker](https://www.docker.com/) and Docker Compose

## Running the broker

```bash
# start RabbitMQ and wait until it is healthy
docker compose up -d --wait

# follow the logs
docker compose logs -f rabbitmq

# stop (the data volume stays)
docker compose down
```

| | |
|---|---|
| Image | `rabbitmq:4.3.4-management-alpine` |
| Container | `marketplace-rabbitmq` |
| AMQP | `localhost:5672` |
| Management UI | <http://localhost:15672> |
| Data | named volume `rabbitmq_data` |
| Health check | `rabbitmq-diagnostics -q ping` every 30 s |

The marketplace's `dev-stack.sh` starts this compose together with the observability stack and the service databases.

### Environment variables

Credentials are read from the environment, with defaults for local development:

| Variable                | Description       | Default |
|-------------------------|-------------------|---------|
| `RABBITMQ_DEFAULT_USER` | RabbitMQ username | `admin` |
| `RABBITMQ_DEFAULT_PASS` | RabbitMQ password | `admin` |

> The defaults are for local use only. Set real credentials through the environment before running this anywhere else.

## Related repositories

| Repository | Description |
|------------|-------------|
| [api-gateway](https://github.com/viniciusferreira7/api-gateway) | Central HTTP entry point for the Marketplace microservices |
| [checkout-service](https://github.com/viniciusferreira7/checkout-service) | Publishes the payment orders |
| [payments-service](https://github.com/viniciusferreira7/payments-service) | Consumes them, owns the retry and dead-letter queues |
