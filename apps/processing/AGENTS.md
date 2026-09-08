# processing — Dossier execution

**Bounded context:** runs each citizen application through the configured Flow.
Port 8001, prefix `/processing/api/v1`. Owns its own DB. This is the default app for
`nest` (root of `nest-cli.json`). Read the root `AGENTS.md` for shared conventions.

## Owns

- **Dossier** (Hồ sơ) — one Citizen's application for one Formality, moving through that
  Formality's Flow. Tracks current step state, who must act next, and history.
- Step **execution state** (pending / in-progress / done per Step), including sequential
  vs. parallel progression as defined by the Flow.

## Boundaries (important)

- Flow/Formality/Step **definitions are owned by `configuring`** — this app consumes them,
  it does not define them. Reference them by id; fetch via the configuring service.
- A Dossier carries the **Citizen** only as a reference (id + snapshot). Citizen identity
  is external (VNeID via `wrapper`) — never create a citizen table here (`docs/adr/0002`).
- Step assignment points at **Units/Officers owned by `organizing`** — reference by id.
- External data/enrichment for a Dossier goes through `wrapper`, not direct outbound calls.

## Specifics

- This is where the **multi-actor workflow logic** lives (sequential/parallel steps across
  units). Keep this logic in focused services — when a service starts handling several
  responsibilities (transition rules, assignment, notifications…), split it
  (Single Responsibility; see root "Code style").

## Don'ts

- ❌ Don't define Formalities/Flows here.
- ❌ Don't store Citizen master data here.
