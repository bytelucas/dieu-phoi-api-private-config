# AGENTS.md — dieu-phoi backend

Backend for **Điều Phối Giải Quyết TTHC**: agencies configure administrative
procedures as multi-step workflows; each citizen application is then routed through
those steps, processed by officers across one or more units, sequentially or in parallel.

For the domain language and bounded contexts, read **[CONTEXT-MAP.md](./CONTEXT-MAP.md)**.
For architectural decisions, read **[docs/adr/](./docs/adr/)**.

## Agent skills

Cấu hình cho bộ skill engineering (`mattpocock/skills`). Cả ba file dưới đây **đều gitignore** —
xem `.gitignore` mục *AI agent config*; chỉ `docs/adr/` được track.

### Issue tracker

**GitLab tự host** `git-ttcpdt.mbfs.vn`, project id **64** — thao tác qua **REST API** + token lấy từ
Git Credential Manager. ⚠️ **`glab` KHÔNG dùng được** (host chưa đăng ký). Xem `docs/agents/issue-tracker.md`.

### Triage labels

Map sang nhãn repo đang dùng — đáng chú ý: `ready-for-agent` → **`ready-for-dev`** (tổ chức **cấm nhãn
chứa chữ `agent`**), `ready-for-human` → **`ready-for-review`**. Xem `docs/agents/triage-labels.md`.

### Domain docs

**Multi-context**: `CONTEXT-MAP.md` ở root + `docs/adr/`; mỗi app trong `apps/` là một bounded context
(**không** phải `src/<context>/` như template mặc định). Xem `docs/agents/domain.md`.

## Architecture

NestJS 11 **monorepo**, pnpm workspaces, Node 22. Each app is an independent
microservice; each owns its **own database** (see `docs/adr/0001`). Shared code lives in
`libs/`.

| App | Port | Prefix | Responsibility |
|-----|------|--------|----------------|
| `gateway` | 8004 | `/gateway/api/v1` | Inbound API gateway / BFF. External callers enter here. |
| `auth` | 8000 | `/auth/api/v1` | Auth & authz for **internal officers only**. |
| `configuring` | 8002 | `/configuring/api/v1` | Defines **Formalities** + their **Flows** (master data). |
| `processing` | 8001 | `/processing/api/v1` | Runs **Dossiers** through the configured Flow. |
| `organizing` | 8003 | `/organizing/api/v1` | Internal **Units** + **Officers**. |
| `wrapper` | 8005 | `/wrapper/api/v1` | Outbound adapter: SSO + external system calls. |

Each app has its own `AGENTS.md` with context-specific rules — read it before working in
that app. `libs/core/AGENTS.md` documents the shared primitives every app builds on.

## Tech stack

NestJS · TypeORM + **Oracle** (`oracledb`) · MongoDB · Redis (`ioredis`) · Pino logging
(`nestjs-pino`) · Swagger · class-validator · pnpm.

## Commands

```bash
pnpm install                       # install deps
pnpm dev <app>                     # run one service with watch (e.g. pnpm dev gateway)
pnpm start <app>                   # run one service, no watch
pnpm debug <app>                   # run with debugger
nest build <app>                   # build one service → dist/apps/<app>/main.js
pnpm lint                          # eslint --fix
pnpm format                        # prettier
pnpm test                          # jest unit
pnpm test:e2e                      # jest e2e

# Code generation — ALWAYS use these, never scaffold by hand
pnpm app:gen <app>                 # new microservice
pnpm module:gen <app> <module>     # new module in an app
pnpm feat:gen <app> <module> <feat># new feature (service + dto + mapper)

# Migrations (TypeORM/Oracle) — run from repo root
pnpm migrate:create <app> <name>   # empty migration
pnpm migrate:gen <app> <name>      # generate from entity diff
pnpm migrate:run <app>             # apply
pnpm migrate:show <app>            # list
pnpm migrate:revert <app>          # roll back last

# Seeds (data scaffolding, NOT migrations) — live in <app>/database/oracle/seed/*.seed.ts,
# each default-exporting (dataSource) => Promise<void>. Kept out of migrations/ so they never
# enter the migrations table. Run manually:
pnpm seed:run <app>                # run every *.seed.ts for an app
```

## Conventions

### Bootstrap
Every app's `main.ts` calls `startApp(AppModule, { appName })` from `@libs/core`, and
`app.module.ts` imports `...commonModules({ appName })`. Don't re-wire global pipes,
interceptors, Swagger, CORS, or versioning per app — `@libs/core` owns all of that.

### Routes & responses
- URI versioning, default `v1`. Full path: `/<app>/api/v1/<route>`.
- **Quy ước API của team (bắt buộc):** mọi endpoint dùng method **POST**, **không dùng GET** (kể
  cả list/detail). Tham số truyền trong **request body**, **KHÔNG param trên path** — không có
  `:id`, không query string. Path theo dạng **`<resource>/<action>`** với action là động từ, ví dụ
  `/role-groups/list`, `/role-groups/create`, `/role-groups/update`, `/role-groups/delete`,
  `/role-groups/set-permissions`, `/permission-catalog/tree`, `/permission-catalog/restore`. Khóa
  (`id`, `roleGroupId`, …) nằm trong body DTO, validate `@IsUuidV7` (không dùng `ParseUuidV7Pipe`
  cho path nữa). Controller đặt `@Controller('')` + đường dẫn đầy đủ trong từng `@Post('/...')` để
  dễ grep. Module mẫu: `user-groups` trong `configuring`, `permission-catalog`/`role-groups`/
  `employee-role-groups` trong `auth`.
- Every response — success and error — carries **two** discriminators: `code` (numeric HTTP
  status) and `statusCode` (a stable **TEXT** code). Clients branch on `statusCode`, not
  `code`. Success is wrapped by `ResponseInterceptor` into
  `{ code: 200, statusCode: 'SUCCESS', message: 'Ok', data }`. To return a raw body (file,
  redirect), annotate the handler/controller with `@SkipResponseEnvelope()`.
- Error body: `{ code, statusCode, message, data, cause, timestamp, path }`. `cause` (the
  underlying caught error) is returned **only in non-production**, stripped to `null` in
  production. See `docs/adr/0004`.
- **Throw errors from a catalog**, never hand-built: `throw CommonErrors.notFound('thủ tục')`
  for framework-level errors, or a domain catalog `throw FormalityErrors.notFound(id, { cause: dbErr })`.
  **All catalogs live in `libs/utils/src/errors/`** (`@libs/utils`) — `CommonErrors` plus every
  domain catalog (e.g. `formality.errors.ts`), shared across services as one error registry.
  Build catalogs with `createErrorFactory({ key: { code: 'UPPER_SNAKE', httpStatus, message } })`
  — `message` is a fixed string or a function interpolating dynamic context; the optional trailing
  `{ cause, data }` carries the original error / extra payload. `feat:gen` does **not** scaffold
  errors — add the catalog to `libs/utils/src/errors/` by hand.
  `AppError extends HttpException`, so the single global `HttpExceptionFilter` (`@Catch()`,
  catch-all) handles everything — including unexpected DB/runtime errors → `500
  INTERNAL_SERVER_ERROR`. Don't build error response objects by hand. Plain
  `throw new HttpException('message_key', HttpStatus.X)` still works (text code derived from
  the status) but the catalog is preferred.
- Swagger is auto-mounted at `/<app>/doc` when `NODE_ENV !== production`. Health check at
  `/<app>/health-check`.

### Code structure (inside an app)
```
apps/<app>/src/
  app.module.ts
  main.ts
  modules/<module>/
    <module>.module.ts
    <module>.controller.ts
    features/
      index.ts            ← AUTO-GENERATED. Never edit by hand.
      <feature>/
        <feature>.service.ts
        <feature>.dto.ts
        <feature>.mapper.ts
```
Error catalogs do **not** live in the app — they're centralized in `libs/utils/src/errors/`
(`CommonErrors` + domain catalogs), see "Routes & responses" above and `docs/adr/0004`.
- **Never hand-create** modules/features or edit `features/index.ts`. Use `module:gen` /
  `feat:gen`. A hook blocks manual edits to `features/index.ts`.
  - This holds **even for dev-only / test-only surfaces** — they are still real modules and must
    live under `modules/<module>/` with the same shape (scaffolded by the generators), not as a
    loose controller at the app `src/` root. Reference implementations: the `configuring`
    `example` module and the `wrapper` `debug` module (both mounted only when
    `NODE_ENV !== production`). Keep the controller **thin** and put logic in a
    `features/<feature>/<feature>.service.ts`; mount the module dev-only from `app.module.ts`.
  - Shared, controller-less **base kits** used by future modules are not modules and may sit at
    the app `src/` root. (Outbound HTTP clients are the exception: the `BaseClient` kit **and** the
    concrete `*Client`s both live in `libs/core/src/http/clients/`, not in the app — a service just
    injects a ready client from `@libs/core`; see `docs/adr/0013`.)
  - **Messaging is infrastructure, not a feature.** A service's concrete Kafka artifacts —
    consumers, publishers, owned-topic specs — live in `apps/<app>/src/message-broker/` (wired by
    `MessageBrokerModule`, which calls `KafkaModule.register({ topics })`), **not** under
    `modules/<module>/features/`. This is the app-root infra-folder pattern above; the shared Kafka
    base kit itself is in `@libs/core` `kafka/`. See `docs/adr/0011`.
- DTOs describe request/response shapes; mappers convert entities → DTOs. `class-validator`
  + `class-transformer` are installed; shared pagination DTOs/types (offset + cursor) live in
  `@libs/utils` (`PaginationOptionQuery`, `PaginationResponse`, cursor variants). The global
  `ValidationPipe` is wired in `@libs/core` (`setupGlobalApp` via `createValidationPipe`):
  `whitelist` + `transform` on, and failures are routed through the error envelope as a 422
  `VALIDATION_FAILED` carrying the flattened `{ field, key, message }[]` in `data` (`field` =
  property path, `key` = constraint name to branch on, `message` = Vietnamese). Annotate DTOs
  with `class-validator` decorators and they validate automatically — no per-app pipe.
- **Validation vocabulary lives in `libs/utils/src/validation/`** (`@libs/utils`), mirroring the
  error catalogs (see `docs/adr/0005`): the central Vietnamese message map for generic
  constraints (`VALIDATION_MESSAGES` — a bare `@IsNotEmpty()` gets its message centrally),
  reusable decorators (`@Trim`, `@IsUuidV7`), the `ParseUuidV7Pipe` for route params, and
  **centralized domain validators** (e.g. `@IsCode` for the master-data `CODE`). Domain
  validators are pure (no DI) and carry their own Vietnamese `message`; DB uniqueness checks
  belong in the service layer, not in a validator. Message precedence is
  `explicit { message } ?? VALIDATION_MESSAGES[key] ?? class-validator default`: for a per-field
  override use a **native** class-validator `{ message }` as the decorator options (e.g.
  `@MaxLength(500, { message: 'Mô tả tối đa 500 ký tự' })`). The flatten layer detects an explicit
  `{ message }` from class-validator's metadata so it wins over the central catalog — no
  `customMessage()` wrapper, and other decorator options (`{ each: true }`, …) keep their plain
  meaning. The flatten + lookup mechanism stays in `libs/core`
  (`exception/validator-exception.ts`). No i18n — Vietnamese only.

### Imports & boundaries
- Use path aliases `@libs/core` and `@libs/utils`. Shared logic goes in `libs/`, not
  copy-pasted between apps.
- **An app must NOT import another app's source** (no `@apps/<other>/*`). In production
  each service is deployed to its **own server/VM**, so another app's code isn't even
  present at runtime — importing it only works by accident at build time. Cross-context
  data is fetched by calling the owning service over HTTP, never by reaching into its code
  or database (see `docs/adr/0001`). Cross-app aliases exist in `tsconfig.json`; share
  contracts via `libs/` types instead of using them.

### Database & entities
- TypeORM + Oracle. Per app: `src/database/oracle/{typeorm.config.ts, oracle.module.ts,
  entities/, migrations/}`. Register via `DatabaseModule.register({ name:
  '<app>_connection', appName, entities })` from `@libs/core`.
- **Entity convention** (see `docs/adr/0003`): Oracle-native. Extend `OracleBaseEntity`
  (the default — adds soft-delete `DELETED_AT`) or `OracleBaseEntityNoSoftDelete` from
  `libs/core`; extend `OracleMasterDataEntity` for master data. **PK = UUID v7 string**
  stored as `VARCHAR2(36)`. Column names `UPPER_CASE`, types `varchar2` / `number` /
  `timestamp`. The base provides audit (`CREATED_AT`, `UPDATED_AT`) and optimistic-locking
  `VERSION` — use `repository.save()` so the version check fires on concurrent updates.
  `OracleMasterDataEntity` also carries `CODE`/`NAME`/`DESCRIPTION`/`STATUS`
  (`MasterDataStatus` = active/inactive).
- **Quan hệ TypeORM chỉ để join, KHÔNG tạo foreign key ở tầng database.** Vẫn khai báo
  `@ManyToOne`/`@JoinColumn` để query join được, nhưng luôn kèm
  `{ createForeignKeyConstraints: false, persistence: false }` — ràng buộc tham chiếu do
  service layer đảm bảo, không phải DB. (Xem `workflow-step.entity.ts` làm mẫu.)
- **Data access via repositories**: extend `BaseTypeOrmRepository<Entity>` (`@libs/core`)
  for query/CRUD — it provides offset pagination (`findAndPaginate`), cursor pagination
  (`findAndCursorPaginate`), safe sortable-field ordering, and `queryRunner`-aware CRUD
  helpers. Place repositories in `<app>/database/oracle/repositories/` and register them.
- Schema changes go through migrations (`migrate:*`), never `synchronize`.

### Config & logging
- Config via `@nestjs/config`, loaded from `libs/core/src/config`. Keys are camelCase
  (e.g. `configuringPort`, `database.configuring`). Read env through `ConfigService`, not
  `process.env` directly (except in `typeorm.config.ts`).
- Logging via Pino (`AppLogger` from `@libs/core`). A request-id is attached per request.

### TypeScript
`strictNullChecks` and `noImplicitAny` are **off** — do not assume strict null safety.
Prefer explicit types anyway; avoid `any`.

### Code style — SOLID, pragmatically
- Apply SOLID, but **don't over-engineer**. Favor code that is easy to read, easy to
  understand, easy to maintain over clever abstractions. No speculative interfaces,
  factories, or layers for things that don't exist yet (YAGNI).
- A **feature service** should do one job. When a service file accumulates too much logic
  (multiple responsibilities, long methods, mixed concerns), **split it** — extract a
  focused service/helper per responsibility — rather than letting one file grow unbounded.
  Single Responsibility is the SOLID principle that matters most here.
- Keep controllers thin (HTTP only); put business logic in services; put entity→DTO
  conversion in mappers.

### Comments
- Viết comment bằng **tiếng Việt**, ngắn gọn và dễ hiểu — ưu tiên giải thích *tại sao*
  (lý do business/kỹ thuật) thay vì mô tả lại code. Tránh comment dài dòng, lan man.
- Giải thích kỹ thuật sâu thì tốt, miễn là vẫn rõ ràng, đi thẳng vào ý.
- Giữ nguyên **thuật ngữ kỹ thuật** ở dạng tiếng Anh (queue, consumer, idempotent, race
  condition, cache, envelope, …) cùng mọi identifier / env-var / config key; chỉ phần diễn
  giải là tiếng Việt — **không** Việt hoá toàn bộ.

### Tài liệu tạm & handoff
- Mọi tài liệu **tạm, không thuộc source** — handoff giữa các phiên, ghi chú điều tra, bản nháp
  phân tích — nằm ở **`.tmp/`** (đã gitignore), **không** để ở `/tmp` hệ điều hành hay scratchpad
  của phiên: phiên sau phải mở lại được, mà scratchpad thì mất theo phiên.
- **Handoff** đặt ở `.tmp/handoff/<YYYY-MM-DD>-<mô-tả-kebab>.md`. Ví dụ đang có:
  `2026-08-06-read-model-abstract-repo-and-mongo-search.md`,
  `2026-08-09-authz-wiring-discovery.md`.
- Handoff **trỏ tới** ADR / MR / commit / file path thay vì chép lại nội dung của chúng; phần đáng
  viết ra là thứ chưa nằm ở đâu cả — quyết định đang chờ, ngã ba đang đứng, cạm bẫy đã vấp.
- Tài liệu thiết kế **chính thức** thì không nằm ở đây: ADR vào `docs/adr/`, design doc vào
  `docs/design/`. `.tmp/` chỉ dành cho thứ sẽ bị vứt đi.

## Git & PRs

- **Đặt tên branch / PR**: `<username>/<base-branch>/<feat|fixbug>/<mã-task-jira>/<description (optional)>`
  — username của người tạo, rồi nhánh đích (thường là `dev`), rồi `feat` cho tính năng mới hoặc
  `fixbug` cho sửa lỗi, rồi mã task Jira, cuối cùng là mô tả ngắn kebab-case (tuỳ chọn).
  Ví dụ: `hieuna/dev/feat/TTHCDC-201/get-list-employee`, `bachtv/dev/fixbug/TTHCDC-163`.
- **Username của user hiện tại là `hieuna`** — dùng nó ở đầu tên branch tạo cho user này.
- **Mã task Jira**: nếu có thì đặt vào đúng vị trí; nếu task chưa có mã Jira thì bỏ qua đoạn đó
  và dùng phần mô tả để nhận diện branch (vd `hieuna/dev/fixbug/search-formality-case-by-formality-id`).
  Không tự bịa mã Jira hay dùng giá trị placeholder.
- **Ngôn ngữ của MR/PR**: **title bằng tiếng Anh** (ngắn, dạng imperative, giữ scope kiểu
  `fix(oracle): ...` nếu có). **Mô tả viết tiếng Việt nhưng KHÔNG Việt hoá toàn bộ** — giữ nguyên
  thuật ngữ kỹ thuật và mọi identifier ở dạng tiếng Anh (repository, migration, cursor pagination,
  IN-list, `queryRunner`, tên file/biến/env). Lý do: title tiếng Anh để đồng bộ với commit/CI và dễ
  scan; mô tả toàn tiếng Việt đọc rất **thô** và dịch thuật ngữ ra tiếng Việt làm người đọc khó hiểu
  hơn chứ không dễ hơn. Cùng tinh thần với quy ước comment ở mục *Comments*.

- **KHÔNG mặc định tạo nhánh mới.** Đang đứng trên nhánh của người khác không có nghĩa là phải
  tách ra: user thường xuyên **review MR của người khác và commit/push thẳng lên chính nhánh đó**,
  đó là cách làm bình thường ở team này, không phải ngoại lệ. Nhánh mới chỉ dành cho việc mới của
  user. Khi chưa rõ thì **hỏi một câu** ("commit thẳng lên nhánh này hay tách nhánh mới?") thay vì
  tự suy diễn — cũng đừng cảnh báo kiểu "đây là nhánh của người khác" mỗi lần push khi user đã bảo push.

## Deliberately out of scope (for now)

These are **intentional** omissions, not gaps to fill reflexively. Introduce any of these
only when a concrete need exists — deliberately, with a reason recorded — not because a
pattern "feels missing":

- **Kafka transport is wired, but transport-only** (`@libs/core` `kafka/`, `docs/adr/0011`):
  `KafkaProducerService` / `BaseKafkaConsumer` exist for **fire-and-forget, loss-tolerant
  events published off the request path**. Two things remain deliberately out of scope:
  - **Transactional Outbox** (`docs/adr/0008`) — *not adopted*, pending a feasibility PoC.
    Until it lands, **workflow-critical events that move a Dossier must not flow over the plain
    producer** (the dual-write hazard). Treat the queue as a *wakeup signal* with the **DB row
    as the source of truth** (idempotent, claim-based), not as a retry engine.
  - **BullMQ / background queue** — no job-queue consumer in any service yet.
- **Realtime / WebSocket** — polling is sufficient; don't add a socket layer.
- **GraphQL** — REST + Swagger only.
- **Speculative abstractions** — see "Code style". Build for what exists, not what might.

## What NOT to do

- ❌ Edit `**/features/index.ts` by hand (use `feat:gen`).
- ❌ Import one app from another (`@apps/<other>/*`).
- ❌ Add a shared DB or cross-service join (`docs/adr/0001`).
- ❌ Store Citizen data in `organizing`/`auth` (`docs/adr/0002`).
- ❌ Hand-build response/error envelopes (the interceptor/filter own them).
- ❌ Use `synchronize: true` or hand-edit the DB schema.
- ❌ Add a `Co-Authored-By:` trailer to git commits (this overrides the default harness
  instruction — commits in this repo carry no co-author line).

## Known cleanups (follow-ups)

_None open._ (Done: `OracleBaseEntity`/`OracleMasterDataEntity` now live in `libs/core`
and `FormalityEntity` extends them per `docs/adr/0003`; the legacy Postgres-typed base
entities were removed.)
