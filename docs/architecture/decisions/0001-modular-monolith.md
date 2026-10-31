# ADR 0001 — Modular monolith with hexagonal boundaries

## Status
Accepted

## Context
Tawla is a ten-day portfolio project needing a single deployable backend that is simple to run
locally and in CI, while still demonstrating clean-architecture discipline. Microservices would add
network partitions, distributed transactions, service discovery, and multiple deployment units that
are disproportionate to the scope. At the same time, a "big ball of mud" monolith would obscure the
domain logic and make the codebase hard to reason about or review.

## Decision
Adopt a **modular monolith with hexagonal (ports and adapters) boundaries**. A single Django backend
is deployed as one unit, but strict package boundaries preserve the option to extract bounded
contexts into services later.

Bounded contexts:
- `accounts` — staff users, authentication, roles, preferred language
- `menu` — restaurants, tables, categories, menu items, availability, QR generation
- `ordering` — orders, order items, state machine, order events, kitchen read models
- `analytics` — sales projections and reporting read models
- `realtime` — WebSocket consumers, channel groups, event dispatch, tickets

Dependency rule (absolute):
```
presentation ──> application ──> domain
                        ^
                        │
                infrastructure (implements application ports)
```

- `domain` depends on nothing but the Python standard library.
- `application` depends on `domain` and defines ports.
- `infrastructure` depends on `application` and `domain` and implements ports.
- `presentation` depends on `application` and the composition root; it never contains business rules.
- `container.py` is the only place allowed to know concrete infrastructure classes.

## Consequences
- Positive: one deployable artifact, simple local development (`docker compose up`), easy to reason
  about, clear import boundaries enforceable via `import-linter` in CI.
- Negative: no independent scaling of bounded contexts; a failure in one context can affect the
  whole process. Acceptable for a single-node portfolio deployment.
- Risk: boundary erosion over time if developers bypass ports. Mitigated by `import-linter` and
  code review.

## Related
- `local-docs/IMPLEMENTATION_PLAN.md §3` Architectural Approach
- `local-docs/IMPLEMENTATION_PLAN.md §7` Repository Layout
- `local-docs/02-tawla-live-orders.md §5` Architecture and Clean Code Rules
