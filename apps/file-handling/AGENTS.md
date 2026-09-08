# file-handling — generic file storage

**Bounded context:** stores and serves **Files** (upload/download now; convert later).
Port **8006**, prefix `/file-handling/api/v1`. Owns its **own MongoDB** (connection
`file-handling_mongo`). Read the root `AGENTS.md` for shared conventions; this file covers
what's specific here.

This context is **deliberately domain-agnostic** — it does not know what a Dossier or
Formality is. The domain meaning of a file ("the CMND scan for Dossier X") belongs to the
linking context (`processing`/`configuring`); here a File is just bytes + metadata.

## Owns

- **File** — a stored object: raw **bytes in MinIO/S3** + an authoritative **metadata**
  record in Mongo (collection `files`). Other contexts hold only a File **id + snapshot**,
  never the bytes.
- The MinIO bucket and the object lifecycle (create now; soft-delete; orphan-sweep later).

### `File` metadata (collection `files`)

`_id` = Mongo's default **ObjectId** (internal, never exposed) · `fileId` = **UUID v7** (the public
id other contexts reference — uniquely indexed) · `bucket` · `objectKey`
(`{yyyy}/{mm}/{dd}/{fileId}{ext}`) · `originalName` (Vietnamese-safe) · `mimeType` · `size`
· `checksum` (sha256 hex) · `status` (`STORED`|`USED` — usage state; `USED` ⇔ claimed, see
`docs/adr/0024`. Deletion is signalled by `deletedAt`, **not** a status value) · `ownerType?`/`ownerId?`
(opaque claim ref) · `createdAt`/`updatedAt`/`deletedAt?` (**epoch-millis numbers**, `docs/adr/0020`
— do **not** use `timestamps:true`, which emits `Date`) · `version` (optimistic lock).

`OracleBaseEntity` is **not** used here (Mongo SoR — `docs/adr/0023`); its guarantees are
re-implemented in the schema: a UUID-v7 **`fileId`** (unique) is the public id while Mongo's `_id`
stays the default ObjectId (mirroring the dvc-api file-service); soft-delete via `deletedAt`;
optimistic concurrency via `version`. There is no `migrate:*` flow — the schema evolves in code.

## Transports (two, by job — `docs/adr/0024`)

- **Bytes (upload/download) → HTTP, frontend-direct.** The frontend calls this service's
  HTTP edge directly (`docs/adr/0016`); `gateway` is not in the path. Bytes **never** go
  over NATS (≤8MB cap, `docs/adr/0017`).
- **Claim (`set-owner`) → NATS only.** `configuring`/`processing` set a File's `ownerRef`
  over NATS after linking it. `ownerRef != null` ⇒ the File is **claimed** (its `status` flips
  `STORED` → `USED` in the same version-checked write, so `status == USED ⇔ ownerRef != null`) and
  the (future) orphan sweeper must spare it.

## HTTP surface (all **POST**, action-style — org rule)

`POST /files/upload` (multipart, **1..N** files, always returns `{ files: File[], failures:
[{ originalName, reason }] }` — **best-effort**, keep the standard envelope) · `POST
/files/download` (`@SkipResponseEnvelope()`, streams bytes; `attachment`|`inline` via
`Content-Disposition`, Vietnamese names via `filename*`) · `POST /files/detail` · `POST
/files/delete` (soft-delete). NATS: `set-owner` handler (no HTTP twin).

## v1 scope — deliberately minimal

- **Upload/download use a buffer** (whole file in RAM) for the first cut — simple to ship.
  The **`MAX_UPLOAD_BYTES` env guard is the safety net** while buffered. Streaming
  (`@fastify/multipart` part stream → `@aws-sdk/lib-storage` `Upload`; `getObject().Body` →
  reply) is the **first follow-up**, done for upload **and** download together.
- **Validation:** per-file size (env) + extension allowlist (env, default
  `pdf,jpg,jpeg,png,doc,docx,xls,xlsx`; empty = allow all) + `X-Content-Type-Options:
  nosniff` and default `attachment` on download.
- **Single bucket** (`MINIO_BUCKET`), **auto-created on boot** (StorageModule
  `onModuleInit`).

### Deferred (intentional — add only on real need)

streaming up/down · presigned PUT/GET (direct-to-MinIO) · AV scan + magic-byte sniff (the
dvc-api MetaDefender `tmp/→scan→stored` gate) · **convert** (derived Files via
`derivedFromFileId`+`variant`, or async jobs — additive, no migration) · orphan-sweeper
cron · NATS `get-files-by-ids` read (build the read **logic** when a concrete caller exists,
wrap NATS then) · officer-JWT auth guard (deferred system-wide while `auth` is a stub,
`docs/adr/0016`).

## Structure

- **S3 client is app-local:** `src/storage/` (`StorageModule` + `S3Client` + a thin
  `S3StorageService`). It is **not** in `libs/core` — only this app uses S3 (the libs/core
  "one-app primitive" rule) and it is not an HTTP `BaseClient` (`docs/adr/0013` N/A).
  `forcePathStyle: true` is required for MinIO.
- **Use the generators** (root `AGENTS.md`): `module:gen file-handling files`, then
  `feat:gen file-handling files upload|download|detail|delete|set-owner`. Thin controller;
  logic in feature services.
- **NATS `set-owner` handler** lives **inside the module** at `modules/files/rpc/` (a sibling of
  `api/`), mirroring `configuring` `modules/example/rpc/example.nats.handler.ts` — `rpc/` is the
  **sync request/reply transport-adapter** folder (NATS req/reply is RPC, docs/adr/0018; NOT async
  Kafka `message-broker/` traffic, which is `processing`-only). Name it `<action>.nats.handler.ts`
  (e.g. `set-owner.nats.handler.ts`), a thin adapter over the `set-owner` **feature service**. The
  request/response wire types are the `@libs/nats` contract used directly — do **not** re-declare them
  as app DTOs (the generator-forced `set-owner.dto.ts` stays an empty stub, like configuring's
  `InternalGetExampleDto`). The controller sits under `modules/files/api/`.
- **App-level shared kit** lives at `src/shared/` (a sibling of `modules/`, mirroring the north-star
  `prediction-market-server/src/shared`), NOT under a module. The **shared public File view**
  (`FileMetadataDto`, `FileIdDto`, `mapFileToMetadataDto` — reused by upload/detail/delete AND referenced
  across transports) lives in `src/shared/dto/`, not in a per-module `dto/` folder.
- **DB:** `DatabaseModule.register` **mongo-only** (no Oracle); `File` schema +
  `FileRepository` in `src/database/mongo/{schemas,repositories}`.
- **Errors:** add a `FileErrors` catalog in `libs/utils/src/errors/` (`notFound`,
  `tooLarge`, `typeNotAllowed`, `uploadFailed`, …) — never hand-build error bodies.

## Don'ts

- ❌ Don't store Dossier/Formality semantics here — `ownerRef` is an **opaque** grouping/
  cleanup hint, not a domain link. No "Dossier ↔ files" join table.
- ❌ Don't send file bytes over NATS, or expose `set-owner` over the public HTTP edge.
- ❌ Don't wire Oracle, `OracleBaseEntity`, or `migrate:*` (Mongo SoR — `docs/adr/0023`).
- ❌ Don't put the S3 client in `libs/core` until a second service needs it.
- ❌ Don't use `timestamps:true` on the schema — store epoch-millis numbers (`docs/adr/0020`).
