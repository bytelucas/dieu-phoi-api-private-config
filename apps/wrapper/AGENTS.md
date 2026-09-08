# wrapper — outbound adapter to external systems

**Bounded context:** everything that **leaves the system**. Port 8005, prefix
`/wrapper/api/v1`. Read the root `AGENTS.md` for shared conventions.

## Role

- **Outbound only**: handles SSO login and calls third-party/national systems to fetch or
  enrich data (e.g. full Citizen info from VNeID/DVCQG — exact partners TBD).
- Acts as the **anti-corruption layer**: translate external payloads into our internal
  shapes so external quirks don't leak into other contexts.

## Direction (don't confuse with gateway)

- `wrapper` = **we call out to others**.
- `gateway` = **others call into us**.

## Specifics

- Citizen identity lives here at the boundary: `wrapper` brings it in via SSO; other
  services receive only a reference (id + snapshot), never a stored citizen master record
  (`docs/adr/0002`).
- Per-integration adapters: when one client/service accumulates several external concerns,
  split per external system (Single Responsibility; see root "Code style").
- "Adapter" here is the **anti-corruption role**, implemented as concrete `*Client` classes
  that **extend `BaseClient`** and live in **`libs/core/src/http/clients/`** — not in this app
  (`docs/adr/0013`). `wrapper` **just injects a ready client from `@libs/core`** (e.g.
  `HttpbinClient`; SSO/VNeID TBD). The `*Client` bakes its base URL + auth headers and maps
  external DTO → internal DTO via `*.mapper.ts`; it is the only place that may catch
  `ExternalSystemError` and translate an upstream status into a domain error
  (`docs/adr/0009`). Outbound transport (timeout, retry, masking, logging, request-id, call-log)
  is inherited from `BaseClient`, not re-implemented per client.

## Don'ts

- ❌ Don't expose inbound public APIs for clients — that's `gateway`.
- ❌ Don't let raw external response shapes flow through unmapped.
