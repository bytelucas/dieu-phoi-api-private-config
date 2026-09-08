# gateway — inbound API gateway / BFF

**Bounded context:** the single entry point external callers use to reach the system.
Port 8004, prefix `/gateway/api/v1`. Read the root `AGENTS.md` for shared conventions.

## Role

- **Inbound only**: receives HTTP from outside and routes/aggregates to the internal services
  (`auth`, `processing`, `configuring`, `organizing`) **over gRPC** (`docs/adr/0014`) — gateway is
  a gRPC client (no gRPC server of its own).
- A Backend-for-Frontend boundary: shapes/aggregates responses for clients. Keep it thin —
  orchestration and aggregation, not business rules.

## Direction (don't confuse with wrapper)

- `gateway` = **others call into us**.
- `wrapper` = **we call out to others** (SSO, external systems).

## Don'ts

- ❌ Don't put domain/business logic here — delegate to the owning service.
- ❌ Don't reach into another app's DB or source; call services over gRPC
  (`docs/adr/0014`; never reach into another DB per `docs/adr/0001`).
- ❌ Don't make outbound third-party calls here — that's `wrapper`.
