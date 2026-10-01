# Sentinel — Project Plan & Build Spec

This document is the source of truth for building Sentinel. Follow it phase by phase,
in order. Do not skip ahead to later phases before the current phase's acceptance
criteria are met. If a decision isn't covered here, ask before assuming.

## 1. What this project is

Sentinel is a distributed transaction orchestration platform that demonstrates reliable
processing across service boundaries during retries and partial failures. It is a
portfolio project, not a toy demo — every feature listed below must be genuinely
implemented, testable, and demoable, because it will be presented as real work in
technical interviews.

**Non-goals:** no frontend/UI. No auth system beyond what's needed to look realistic
(if anything). No feature not listed in this plan — scope creep kills side projects.

## 2. Tech stack (fixed — do not substitute without asking)

- Java 21, Spring Boot 3.3.x, Maven
- PostgreSQL (one instance, three databases: `order_db`, `payment_db`, `inventory_db`)
- Kafka-API compatible broker: Redpanda (via Docker Compose)
- Resilience4j for circuit breakers / retry / backoff
- OpenTelemetry + Prometheus + Grafana for observability
- Docker + Docker Compose for local dev; target deployment is a single free-tier host
  (Railway/Fly.io/Render) or a minimal AWS setup — decide in Phase 5, not before
- Lombok, JUnit 5, Testcontainers for integration tests where practical

## 3. Services and their responsibilities

| Service | Port | Owns | Responsibility |
|---|---|---|---|
| `order-service` | 8081 | `order_db` | Accepts orders, writes outbox events, publishes `OrderCreated` |
| `payment-service` | 8082 | `payment_db` | Consumes `OrderCreated`, processes payment idempotently, publishes `PaymentSucceeded`/`PaymentFailed` |
| `inventory-service` | 8083 | `inventory_db` | Consumes payment result, reserves/releases stock, publishes `InventoryReserved`/`InventoryReleased` |

Repo layout is already scaffolded as a monorepo:
```
sentinel/
├── order-service/
├── payment-service/
├── inventory-service/
├── infra/init-db.sql
├── docker-compose.yml
└── README.md
```

## 4. Event contract (define before writing consumers)

Each event is a JSON payload on its own Kafka topic. Minimum required fields:

- `OrderCreated` — orderId, customerId, items[], totalAmount, timestamp
- `PaymentSucceeded` / `PaymentFailed` — orderId, paymentId, amount, reason (if failed), timestamp
- `InventoryReserved` / `InventoryReleased` — orderId, items[], timestamp

Every event carries a `correlationId` (= orderId is fine) so a full transaction can be
traced across all three services' logs and traces.

## 5. Phase 1 — Outbox pattern + idempotency core

**Goal:** prove the core reliability primitive before building anything else on top of it.

Tasks:
1. `order_db` schema: `orders` table + `outbox_events` table (id, aggregate_id, event_type, payload JSONB, created_at, published_at nullable).
2. `POST /orders` endpoint: single `@Transactional` method that inserts the order row and the outbox row together.
3. Scheduled poller (`@Scheduled`, ~500ms–1s) that reads unpublished outbox rows, publishes to Kafka, marks `published_at`.
4. `payment_db` schema: `processed_events` table (event_id PK, processed_at).
5. Kafka listener in `payment-service` for `OrderCreated`: check `processed_events` before doing any work; insert into it in the same transaction as any payment write.
6. Tests: (a) force a failure after the order insert and prove no orphaned outbox row exists; (b) replay the same event twice and prove payment isn't double-processed.

**Definition of done:** placing an order results in exactly one payment attempt, even if
the `OrderCreated` message is delivered twice. Both proven by an automated test, not
manual observation.

## 6. Phase 2 — Full saga workflow

Tasks:
1. `payment-service` publishes `PaymentSucceeded`/`PaymentFailed` (simulate the payment provider — no real integration needed, use a configurable success/failure rate).
2. `inventory-service` consumes payment result, reserves stock on success, and — this is the important part — implements the **compensating action**: on `PaymentFailed`, no reservation happens; if inventory itself fails after a successful payment, publish a compensation event that triggers a refund flow back in `payment-service`.
3. Extend `order_db` with an order status field that reflects the saga's current state (`PENDING`, `CONFIRMED`, `FAILED`, `COMPENSATING`, `CANCELLED`).
4. Order status is updated as terminal events flow back to `order-service` (it should also consume events, not just publish).

**Definition of done:** you can trigger a failure at the inventory stage and watch the
system correctly unwind the payment via the compensating event, ending in a consistent
`CANCELLED` state — not a stuck or contradictory one.

## 7. Phase 3 — Resilience mechanisms

Tasks:
1. Wrap inter-service HTTP calls (if any exist) and the simulated payment provider call with Resilience4j circuit breaker + retry with exponential backoff and jitter.
2. Add basic load shedding: reject/queue new order requests past a configured concurrency threshold, returning 429/503 rather than degrading silently.
3. Configure a dead-letter topic per consumer; after N failed processing attempts, route the message there instead of blocking the consumer.
4. Build a small replay mechanism (admin endpoint or CLI) that reads from the DLQ and reprocesses messages on demand.
5. Tests/demo scenario: kill `payment-service` mid-flow, show orders queue/retry correctly, restart it, show it catches up including any DLQ replay.

**Definition of done:** you can kill any one service, generate load against the others,
and the system degrades gracefully (no data loss, no silent drops) and recovers cleanly
when the service comes back.

## 8. Phase 4 — Observability

Tasks:
1. Add OpenTelemetry instrumentation to all three services; propagate `correlationId`/trace context through Kafka message headers, not just HTTP.
2. Export metrics to Prometheus: request rate, error rate, DLQ depth, saga completion time.
3. Stand up Grafana with a dashboard showing the above, wired to the Prometheus source.
4. Run a load test (k6 or similar) to generate real traffic, and capture actual latency/throughput numbers — these replace any placeholder numbers in the README and resume.

**Definition of done:** a single trace can be followed end-to-end across all three
services for one order, and the Grafana dashboard shows real numbers from an actual
load test, not estimates.

## 9. Phase 5 — Deployment

Tasks:
1. Decide the deployment target (default assumption: Railway or Fly.io for simplicity, given this is a solo demo project — confirm before spending time on AWS ECS setup, which is more work for similar payoff here).
2. Containerize all three services with production-appropriate Dockerfiles (multi-stage builds, non-root user, slim base image).
3. Deploy Postgres, Redpanda (or a managed Kafka-compatible service if the host offers one), and all three services.
4. Confirm the live system is reachable and can process a real order end-to-end from an external client (e.g. a `curl` command anyone could run).

**Definition of done:** a public URL exists that a stranger could hit to trigger an order
and observe the flow, without needing your local machine running.

## 10. Phase 6 — Documentation and resume alignment

Tasks:
1. Write the full README: architecture diagram, the failure scenarios handled, how to run locally, link to the live demo, and the real metrics from Phase 4's load test.
2. Record a short demo (GIF or video) of a request flowing through the system and the Grafana dashboard under load; embed or link it in the README.
3. Update the resume bullets for Sentinel so every claim maps to something demoable — swap any estimated metric for the real load-test number.
4. Prepare a 2-minute verbal walkthrough of the architecture — this is what actually gets used in the interview, the code is the proof behind it.

**Definition of done:** every bullet on the resume for this project can be defended by
pointing at a specific piece of code, a specific test, or a specific dashboard panel.

## 11. Working agreement for whichever LLM is executing this plan

- Work one phase at a time. Do not start Phase N+1 until Phase N's "Definition of done" is met.
- Prefer working, tested code over more features. A finished Phase 3 beats a half-built Phase 5.
- When a design choice isn't specified here (e.g. exact retry counts, exact table indexes), make a reasonable default choice and note it, rather than stopping to ask — but flag anything that changes the tech stack, service boundaries, or event contract in Section 2–4, since those are load-bearing for the rest of the plan.
- Write a short commit per meaningful unit of work, not one giant commit per phase.
- Keep the README's "Status" and "Roadmap" sections current as phases complete.
