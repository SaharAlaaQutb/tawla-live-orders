# ADR 0003 — Short-lived signed tickets for WebSocket auth

## Status
Accepted

## Context
Browser WebSockets cannot set arbitrary HTTP headers, so staff JWTs cannot be passed the same way
as REST requests. Putting long-lived JWTs in WebSocket query strings leaks them into server logs,
proxy logs, and browser history. Kitchen and tracking sockets need scoped, short-lived
authorization without exposing long-lived credentials.

## Decision
Issue **short-lived signed tickets** for WebSocket authentication:

1. Client calls `POST /api/realtime/tickets/` with staff credentials (JWT) or an order tracking token.
2. Server returns a signed ticket valid for 60 seconds with explicit scopes:
   - `kitchen:read`, `kitchen:write` for staff kitchen sockets.
   - `order:read` bound to one order id for customer tracking sockets.
   - `menu:read` bound to one restaurant for customer menu sockets.
3. Client connects with `?ticket=...`.
4. The consumer verifies the ticket signature, expiry, scope, and subject **before** joining any
   channel group. Consumers never join a group from raw client input.

## Consequences
- Positive: long-lived JWTs never travel over WebSocket query strings; tickets are short-lived and
  scope-bound, limiting blast radius if intercepted; consumers authorize before any group join.
- Negative: an extra HTTP round-trip before each WebSocket connection; ticket issuance endpoint
  must be rate-limited and itself authenticated.
- Risk: ticket theft within the 60-second window. Mitigated by short lifetime, HTTPS/WSS only, and
  scope binding (a stolen ticket only grants the scoped subject it was issued for).

## Related
- `local-docs/IMPLEMENTATION_PLAN.md §8.6` Real-time design — Authentication
- `local-docs/IMPLEMENTATION_PLAN.md §6.4` Channel groups
- `local-docs/02-tawla-live-orders.md §4` Tech Stack — Django Channels
