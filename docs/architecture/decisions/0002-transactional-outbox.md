# ADR 0002 — Transactional outbox for domain events

## Status
Accepted

## Context
Domain events (`OrderPlaced`, `OrderStatusChanged`, `MenuItemAvailabilityChanged`) must be
published to WebSocket subscribers (kitchen board, customer tracking) only after the corresponding
database transaction commits. Publishing inside the transaction risks notifying subscribers about
an order that later rolls back. Publishing after commit but outside the transaction risks losing
events if the process crashes between commit and publish.

## Decision
Implement a **transactional outbox**. Domain events are collected by the aggregate, drained by the
use case, and appended to an `order_events_outbox` table in the **same transaction** as the
aggregate state change. A background dispatcher polls the outbox for unpublished events, publishes
them to the Redis channel layer, and marks them published. Retries use exponential backoff; the
dispatcher is idempotent because clients key updates by `event_id` and `order.version`.

Flow:
1. Use case executes within a `UnitOfWork` transaction.
2. Aggregate state and outbox rows are saved atomically.
3. Transaction commits.
4. Dispatcher fetches unpublished outbox rows.
5. Dispatcher publishes to channel groups (`kitchen.{restaurant_id}`, `order.{order_id}`, etc.).
6. Dispatcher marks rows as published.

## Consequences
- Positive: at-least-once delivery guarantee; events survive a crash between commit and publish;
  kitchen never sees an uncommitted order; no distributed transactions needed.
- Negative: eventual (not immediate) delivery; slight latency between commit and subscriber
  notification; outbox table requires maintenance (cleanup of old published rows).
- Risk: duplicate delivery if the dispatcher crashes after publishing but before marking. Mitigated
  by client-side deduplication on `event_id` and `order_version`.

## Related
- `local-docs/IMPLEMENTATION_PLAN.md §3.3` Consistency and transaction boundaries
- `local-docs/IMPLEMENTATION_PLAN.md §6.2` Order placement event flow
- `local-docs/IMPLEMENTATION_PLAN.md §6.3` Why publish after commit
- `local-docs/IMPLEMENTATION_PLAN.md §5` Design Patterns — Transactional Outbox
