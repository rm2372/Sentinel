# Sentinel — Distributed Transaction Orchestration Platform

A distributed transaction orchestration platform demonstrating reliable processing across
service boundaries using the transactional outbox pattern, idempotent consumers, and
Kafka-based event workflows — with resilience mechanisms (circuit breakers, backoff with
jitter, dead-letter queue replay) and full observability (OpenTelemetry, Prometheus, Grafana).

## Architecture

```
order-service (8081) --outbox--> Kafka/Redpanda --> payment-service (8082)
                                                  --> inventory-service (8083)
```

*(diagram to be expanded as the saga flow is built out)*

## Services

| Service | Port | Responsibility |
|---|---|---|
| order-service | 8081 | Accepts orders, writes to outbox, publishes OrderCreated |
| payment-service | 8082 | Consumes OrderCreated, processes payment idempotently |
| inventory-service | 8083 | Consumes payment result, reserves/releases stock |

## Running locally

```bash
docker compose up -d          # Postgres (order_db, payment_db, inventory_db) + Redpanda
cd order-service && ./mvnw spring-boot:run
cd payment-service && ./mvnw spring-boot:run
cd inventory-service && ./mvnw spring-boot:run
```

Redpanda Console (inspect topics/messages): http://localhost:8090

## Status

🚧 Under active development. Current milestone: transactional outbox + idempotent
consumer core.

## Roadmap

- [ ] Outbox pattern + idempotency (order-service → payment-service)
- [ ] Full saga workflow across all three services
- [ ] Resilience: circuit breakers, backoff with jitter, DLQ + replay
- [ ] Observability: OpenTelemetry tracing, Prometheus metrics, Grafana dashboards
- [ ] Deployment
