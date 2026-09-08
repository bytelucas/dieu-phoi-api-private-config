# libs/core — shared platform primitives

Every app builds on `@libs/core`. Change things here with care: a change affects **all six
services**. Read the root `AGENTS.md` first for the conventions these primitives enforce.

## What lives here

- `app/boot.ts` — `startApp(AppModule, { appName })`: creates the Nest app, sets global
  prefix `<app>/api`, URI versioning (default v1), CORS, Pino logger, Swagger (non-prod),
  and the global interceptors/filter. This is the single bootstrap path — apps must not
  reconfigure these themselves.
- `app/common-modules.ts` — `commonModules({ appName })`: ConfigModule, AppLogger, Http.
  Every app spreads this into its `app.module.ts` imports.
- `app/response.ts` — `ResponseInterceptor` → `{ code: 200, message: 'Ok', data }`.
- `app/skip-response-envelope.decorator.ts` — `@SkipResponseEnvelope()` to bypass it.
- `exception/http-exception.filter.ts` — normalizes thrown `HttpException`s into the error
  envelope by message key. (≥500 responses carry `statusCode` = the HTTP status code; <500
  responses carry `code` = the in-body code — this distinction is intentional.)
- `database/` — `DatabaseModule.register(...)`, the Oracle base entities, and
  `BaseTypeOrmRepository` (offset + cursor pagination, safe ordering, CRUD helpers).
- `caching/` — Redis (ioredis) primitives. `CachingService` (raw key/value: get/set/del/setnx,
  hashes — auth sessions, locks, flags), `createRedisClient` (connection factory: retry + event
  logging), and **`RedisQueryResultCache`** — a plain TypeORM `QueryResultCache` (no
  stale-while-revalidate). Enable per query with `.cache({ id, milliseconds })` and invalidate on
  write via `dataSource.queryResultCache.remove([id])`; wired in `DatabaseModule.register`.
  **Read `docs/adr/0009` before touching caching.**
- `health/` — readiness probe soi dependency (`docs/adr/0034`). `HealthModule.register({ checks })`
  được **từng app** khai tường minh trong `app.module.ts`; một check = **một connection** (không phải
  một engine), và `required` — hỏng thì lật readiness hay chỉ báo `degraded` — là thuộc tính của app,
  khai ngay tại đó. Route `health/ready`; `health-check` của `DefaultRouteController` **giữ nguyên**
  hằng `'OK'` cho liveness + startup, đừng cho nó soi dependency.
- `config/` — `@nestjs/config` loader + `AppConfig` type. Keys are camelCase.
- `logger/` — Pino logger, factory, request-id integration.
- `middleware/request-id.middleware.ts` — attaches a request id and opens the ALS context.
- `context/` — `requestContext`, a tiny AsyncLocalStorage carrying per-request values
  (`requestId`) to **singleton** services without request-scoping them (`docs/adr/0010`).
- `kafka/` — **transport-only** Kafka primitives (`docs/adr/0011`): `KafkaModule.register(...)`,
  `KafkaProducerService` (idempotent producer, **nén `zstd`** — wire codec vẫn là JSON, xem
  `docs/design/kafka-codec-va-xu-ly-loi.md` §3; `emit` builds the envelope, auto-stamps `requestId`
  from the ALS context — **fire-and-forget, off the request path** only), `BaseKafkaConsumer`
  (per-handler consumer group, **hai đường xử lý lỗi** poison→DLQ / systemic→`pause`+`resume` chờ vô
  hạn — `docs/adr/0036`, commit-after-success), and the
  `KafkaTopicProvisioner` (owner-declared topics created at startup). No Outbox — see `0008`/`0011`.
  ⚠️ **Idempotency tầng nghiệp vụ là LUẬT với mọi consumer** (`docs/adr/0036` §5): đường systemic giao
  lại cùng một message hàng chục lần. Consumer mới không có chống trùng thì không merge — checklist 4
  câu ở §5 của ADR đó.

## Rules

- Keep this library **framework-level and domain-free**. No Formality/Dossier/Officer
  logic here — that belongs to the apps. If a primitive is only used by one app, it does
  not belong in `libs/core`.
- **Base entities are the canonical entity convention** (`docs/adr/0003`): extend
  `OracleBaseEntity` (ID + audit + optimistic-lock `VERSION` + soft delete) for normal
  tables, `OracleBaseEntityNoSoftDelete` for append-only/stats tables, or
  `OracleMasterDataEntity` (adds `CODE`/`NAME`/`DESCRIPTION`/`STATUS`) for master data.
  Oracle-native, UUID v7 PK as `VARCHAR2(36)`, `UPPER_CASE` columns. `FormalityEntity` is the
  reference implementation. Repositories extend `BaseTypeOrmRepository<Entity>`.
- Anything exported from here must be re-exported through the relevant `index.ts` and
  reachable via the `@libs/core` alias.
