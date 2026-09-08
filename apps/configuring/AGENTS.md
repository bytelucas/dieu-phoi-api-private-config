# configuring — Formality & Flow definitions

**Bounded context:** master data for *how* administrative procedures are processed.
Port 8002, prefix `/configuring/api/v1`. Owns its own Oracle DB (connection `configuring_oracle`)
and a Mongo DB (connection `configuring_mongo`) — names derived from `appName` (docs/adr/0007).
Read the root `AGENTS.md` for shared conventions; this file covers what's specific here.

## Owns

- **Formality** (Thủ tục hành chính) — a defined administrative procedure. Current entity:
  `database/oracle/entities/formality.entity.ts`.
- **Flow** (quy trình) — the ordered/parallel sequence of **Steps** a Formality follows,
  which may differ per agency/unit.
- **Step** — one unit of work in a Flow; references the **Unit** (owned by `organizing`)
  that must act, and whether it runs sequentially or in parallel with siblings.

This context is the **source of truth for definitions only**. It does not run Dossiers —
`processing` consumes these definitions and executes them.

## Specifics

- This is the only app with a live DB setup today — it's the **reference pattern** for
  adding Oracle to other apps: `database/oracle/{typeorm.config.ts, oracle.module.ts,
  entities/, migrations/}`, registered via `DatabaseModule.register({ appName: APP_NAME,
  entities })`. Connection names come from `database/connection.ts`
  (`ORACLE_CONNECTION` / `MONGO_CONNECTION`); repositories inject by those constants
  (`@InjectDataSource(ORACLE_CONNECTION)` / `@InjectModel(name, MONGO_CONNECTION)`) per
  `docs/adr/0007`.
- New entities follow `docs/adr/0003` (Oracle-native, UUID v7 PK). `FormalityEntity` now extends
  `OracleMasterDataEntity` (UUID v7 PK + CODE/NAME/DESCRIPTION/STATUS), so it is the canonical
  reference — the earlier numeric-PK realignment is done.
- Schema changes: `pnpm migrate:gen configuring <name>` → review the generated Oracle SQL
  → `pnpm migrate:run configuring`.

## Don'ts

- ❌ Don't put Dossier/execution state here — that's `processing`.
- ❌ Don't reference Officer/Unit by importing `organizing`; reference Units by id.
