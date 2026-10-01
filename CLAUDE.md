# Sentinel — Project Context for Claude Code

This file is read automatically at the start of every Claude Code session in this
repo. It exists so you don't have to re-explain the project each time.

## What this is

A distributed transaction orchestration platform (portfolio project) demonstrating
the transactional outbox pattern, idempotent consumers, Kafka-based sagas, resilience
mechanisms, and observability across three Spring Boot microservices.

**Full build plan and phase-by-phase acceptance criteria: see `PLAN.md`.** Always
check which phase is current before starting work, and don't move to the next phase
until the current one's "Definition of done" is met.

## Architecture

- `order-service` (8081) — owns `order_db`. Accepts orders, writes outbox events.
- `payment-service` (8082) — owns `payment_db`. Consumes `OrderCreated`, idempotent.
- `inventory-service` (8083) — owns `inventory_db`. Reserves/releases stock.
- Postgres + Redpanda (Kafka-API compatible) via `docker-compose.yml`.

## Conventions

- Java 21, Spring Boot 3.3.x, Maven. Lombok for boilerplate reduction.
- Package structure per service: `com.sentinel.<service>.{controller,service,repository,model,event,config}`.
- Every table that participates in cross-service consistency (outbox, processed_events)
  gets a migration file, not `ddl-auto: update` in the long run — Phase 1 uses
  `ddl-auto: update` for speed, but plan to move to Flyway before Phase 5 (deployment).
- Every consumer must be idempotent before it's considered done — no exceptions, this
  is the core thing this project proves.
- Commit style: small, meaningful commits per unit of work (e.g. "Add outbox table and
  transactional order write"), not one commit per phase.
- No frontend. No dependency not already listed in `PLAN.md` Section 2 without asking first.

## Commands

```bash
docker compose up -d                          # start Postgres + Redpanda
cd order-service && mvn spring-boot:run       # run a service
mvn test                                      # run tests for a service (run from its folder)
```

## Working agreement

- Work one phase of PLAN.md at a time. Use plan mode for anything that touches more
  than one service or changes the event contract, so the approach is visible before
  code gets written.
- Prefer the `code-reviewer` subagent after any non-trivial change, and the
  `test-runner` subagent to run and summarize test output rather than dumping full
  Maven logs into the main conversation.
- Flag (don't silently decide) anything that changes: the tech stack, a service
  boundary, or an event contract field in PLAN.md Section 4 — those are load-bearing
  for the rest of the plan.
