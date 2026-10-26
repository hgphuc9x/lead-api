# Backend Architecture Documentation

## 1. System Overview

For project-specific rules including feature structure, naming conventions, and development guidelines, please refer to [BE-PROJECT-RULES.md](./BE-PROJECT-RULES.md). Data model: [DATABASE.md](./DATABASE.md). API contract: [API_SPEC.md](./API_SPEC.md).

This document focuses on the architectural aspects, communication patterns, and technical implementation details of the sales-lead-management backend (backend only; Swagger UI is the client).

### Key Characteristics:
- **NestJS** with TypeScript, **modular monolith** (modules can be split into services later)
- **PostgreSQL** as the source of truth with **TypeORM**; **Redis** (single instance) for token blacklist, rate limiting and lead cache
- **Stateless API**: any number of instances behind a load balancer
- **RESTful API** with JSON bodies, versioned under `/api/v1`
- **JWT Authentication** (access + refresh) plus an **API key** for the dealership website
- **Three roles**: ADMIN (accounts), MANAGER (whole dealership), SALER (own assigned leads)
- **Multi-tenant ready**: every row belongs to a dealership; every query is scoped by it

### System Components:
```
Dealership website ─┐  (API key)
                    ├──► Load balancer ──► NestJS API (N instances) ──► PostgreSQL primary ⇄ standby replica
Swagger UI ─────────┘  (JWT)                      │
                                                  ├──► Redis (blacklist, cache, rate limit)
                                                  └──► /metrics ◄── Prometheus ──► Grafana
```

## 2. Request Flow

```
Request → Middleware → Guards → Pipes → Controller → Service → Repository → PostgreSQL
                                                         │
                                                         └──► Redis (cache / blacklist)
Response ← Interceptors ← Exception Filter (RFC 7807)
```

### Layer Responsibilities:
- **Middleware**: requestId (`X-Request-Id`), request logging
- **Guards**: `JwtAuthGuard` (signature, expiry, Redis blacklist), `ApiKeyGuard` (website), `RolesGuard` (ADMIN / MANAGER / SALER), `ThrottlerGuard`
- **Pipes**: `ValidationPipe` + DTOs, `ParseUUIDPipe`
- **Controllers**: HTTP routing, service delegation, Swagger decorators
- **Services**: business logic, role scoping, transactions, cache orchestration
- **Repositories (TypeORM)**: data access, isolate DB details from business logic
- **Interceptors**: metrics, timeout
- **Exception Filter**: converts any error to RFC 7807 with `traceId`

### Example Flow - Add Activity (`POST /leads/:id/activities`):
1. `JwtAuthGuard` validates the token and checks the Redis blacklist; `RolesGuard` allows MANAGER and SALER
2. `ValidationPipe` validates `CreateActivityDto`
3. `LeadsService.addActivity` checks the lead is in scope (`dealership_id`, and `assignee_id = userId` for SALER), then opens `dataSource.transaction()`:
   - `INSERT` the activity
   - `UPDATE leads SET status = 'CONTACTED', version = version + 1 WHERE id = ? AND status = 'NEW'` (0 rows is fine)
   - `COMMIT` (on error `ROLLBACK`, cache untouched)
4. After COMMIT: re-read the lead and the 20 latest activities, then `SET` both `lead:{dealershipId}:{leadId}` and `lead:{dealershipId}:{leadId}:activities` (on Redis failure: log a warning and `DEL` the keys)
5. Return `201` with the activity

### Example Flow - Lead Detail (two API calls from the UI):
1. `GET /leads/:id`: guard → `GET lead:{dealershipId}:{leadId}` in Redis
   - **Hit**: scope check using the cached `assigneeId`, return the lead, DB is skipped
   - **Miss**: read the lead (filtered by `dealership_id`) from PostgreSQL, `SET` cache with TTL (404 is never cached)
2. `GET /leads/:id/activities` (default `offset=0, limit=20, order=desc`): `GET lead:{dealershipId}:{leadId}:activities`
   - **Hit**: scope check with the cached `assigneeId`, return the page
   - **Miss**: check the lead is in scope, read the page with `LIMIT/OFFSET`, `SET` cache with TTL
   - Any other `offset`, `limit` or `order` reads PostgreSQL directly and is not cached

### Example Flow - Update Lead (`PATCH /leads/:id`):
1. Validate the DTO: `version` required, at least one of `status` / `assigneeId`
2. Role check: a SALER sending `assigneeId` gets `403`; a SALER may only change leads assigned to them
3. Validate the transition (`422 INVALID_STATUS_TRANSITION`) and the assignee (`422 INVALID_ASSIGNEE`)
4. `UPDATE leads SET ..., version = version + 1 WHERE id = ? AND dealership_id = ? AND version = ?`; 0 rows → `404` if the lead is gone, else `409 LEAD_VERSION_CONFLICT`
5. After COMMIT: `DEL lead:{dealershipId}:{leadId}` (and `...:activities` when `assigneeId` changed)

## 3. Cross-Feature Communication

### Allowed Methods:
- **Module Imports (NestJS DI)**: LeadsModule imports UsersModule (validate assignee); AuthModule imports UsersModule (credentials, deactivate)
- **Dependency Injection**: services inject other modules' exported services through constructors
- **Global modules**: `RedisModule`, `ConfigModule`, `MetricsModule` are global
- **Events**: NestJS EventEmitter is not used now; it is the planned option for async side effects (e.g. notify a saler on a new lead)

### Forbidden Practices:
- Direct import of another feature's internal file: `import { UsersService } from '../users/users.service'` ❌
- Use the exported service via the module: `constructor(private usersService: UsersService)` ✅

### Feature Dependencies (as defined in BE-PROJECT-RULES.md):
- **auth**: depends on users, redis
- **users**: standalone
- **dealerships**: standalone
- **leads**: depends on users, redis
- **health**: depends on typeorm, redis
- **metrics**: standalone
- **common / config**: shared utilities used by all features

## 4. Data Flow & Usage Patterns

1. **Lead ingest**:
   - Website sends `POST /leads` (API key) with optional `Idempotency-Key` (UUID v4, one per form submission)
   - Flow: SELECT by `(dealership_id, idempotency_key)`; found → `200` with the existing lead; not found → INSERT (`201`). The partial unique index is the final guard against races
   - No key means a new lead every time; there is no dedup by name/email
2. **Authentication**:
   - Login validates the bcrypt hash and inserts a `refresh_tokens` row (SHA-256 hash, `device_info`); one row per device, so multi-device login works
   - Access token 15 min, refresh token 7 days
   - Refresh rotates: the old row gets `revoked_at`, a new row is inserted. Reuse of a revoked token revokes all tokens of that user
3. **Revocation**:
   - Logout sets `revoked_at` on that device's row and blacklists the access token (`bl:jti:{jti}`)
   - Deactivating an employee sets `is_active = false` and `deactivated_at`, revokes all their refresh tokens and writes `bl:user:{userId}` so existing access tokens are rejected
4. **Lead lifecycle**: `NEW → CONTACTED → QUALIFIED → WON / LOST` (WON/LOST only from QUALIFIED); the first activity moves `NEW → CONTACTED` automatically
5. **Concurrency**:
   - Concurrent activities: no deadlock (INSERT takes `KEY SHARE`, UPDATE takes `NO KEY UPDATE`, they do not conflict); keep the lock order `INSERT activity → UPDATE lead`; retry once on `40P01` / `40001`
   - Manual edits (several managers at once): optimistic lock via `version`; the later writer gets `409` and must reload
6. **Role-based access**:
   - ADMIN: creates and deactivates accounts, no lead access (`403`)
   - MANAGER: all leads of the dealership; assigns leads and changes status
   - SALER: only leads assigned to them; may change status, not the assignee
   - A lead outside the caller's scope returns `404`

## 5. Configuration Management

### Environment Variables (.env):
- **PostgreSQL**: DATABASE_URL (or DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_NAME)
- **Redis**: REDIS_URL
- **JWT**: JWT_ACCESS_SECRET, JWT_ACCESS_EXPIRES_IN (15m), JWT_REFRESH_SECRET, JWT_REFRESH_EXPIRES_IN (7d)
- **Ingest**: INGEST_API_KEYS (mapping API key → dealership)
- **Cache / limits**: LEAD_CACHE_TTL_SECONDS, THROTTLE_TTL, THROTTLE_LIMIT
- **Observability**: LOG_LEVEL, OTEL_SERVICE_NAME, OTEL_EXPORTER_OTLP_ENDPOINT (optional)
- **Application**: PORT, NODE_ENV

### Configuration Structure:
- Uses `@nestjs/config` with `ConfigModule.forRoot({ isGlobal: true })` and schema validation at startup
- Access via ConfigService injected into services/providers

### Security Practices:
- **Never commit** .env to git; keep `.env.example` as a template
- Passwords stored as bcrypt hashes; refresh tokens as SHA-256 hashes
- Email uniqueness enforced at database level
- TLS terminated at the load balancer; `/metrics` reachable from the internal network only
- Customer PII and activity notes are redacted from logs and never used in metric labels or trace attributes
- `dealership_id` always comes from the authenticated principal (JWT or API key), never from the request

## 6. Validation & Error Handling

### Validation:
- **DTO Validation**: class-validator with global ValidationPipe
- **Transformation**: class-transformer for type coercion (query numbers, dates)
- **Custom Validation**: status transition rules and assignee check live in the service

### Error Handling:
- **Built-in Exceptions**: NotFoundException, ConflictException, UnauthorizedException, ...
- **Custom Exceptions**: extend HttpException with a named error code (`LEAD_NOT_FOUND`, `LEAD_VERSION_CONFLICT`, `INVALID_STATUS_TRANSITION`, ...); full list in API_SPEC.md section 5
- **Global Filter**: HttpExceptionFilter in `common/filters` returns RFC 7807 with `traceId`
- **Database errors**: unique violation on the idempotency index is handled as a replay; deadlock/serialization errors (`40P01`, `40001`) are retried once

## 7. Response Format Standards

See [API_SPEC.md](./API_SPEC.md) section 4: resource as-is for success, `{ items, nextCursor }` for the lead inbox, `{ items, offset, limit, hasMore }` for activities and users, RFC 7807 for errors.

## 8. TypeORM Best Practices

- **Entities** in `entities/` with `@Entity`; columns mapped to snake_case names
- **Migrations**: versioned, `synchronize: false`; partial unique index and CHECK constraints created in a migration
- **Timestamps**: `@CreateDateColumn` / `@UpdateDateColumn` with `timestamptz`
- **Indexes**: declared with `@Index` and listed in DATABASE.md; review the query plan for list queries
- **Transactions**: `dataSource.transaction()` for multi-table writes; keep them short and make no external calls inside
- **Locking**: `@VersionColumn` + conditional UPDATE (QueryBuilder updates must set `version = version + 1` explicitly); no pessimistic locks unless justified
- **Queries**: QueryBuilder for cursor pagination of the inbox; `LIMIT/OFFSET` (fetch `limit + 1`) for activities and users; select only the columns needed for lists; avoid N+1
- **Connections**: pool sized per instance; PgBouncer when scaling out

## 9. Observability

- **Logs**: nestjs-pino JSON to stdout with `timestamp`, `level`, `msg`, `requestId`, `traceId`, `userId`, `dealershipId`, `route`, `statusCode`, `durationMs`
  - `error`: 5xx, abnormal 4xx, rate limit, rejected tokens
  - `warn`: an incident while the request still works (Redis down → DB fallback, slow request)
  - `info`: normal requests, including 400 and 404
  - `debug`: development only
  - Redacted: email, phone, refresh token, password, note (and `authorization`)
- **Metrics** (`/metrics`, Prometheus + Grafana): `http_requests_total`, `http_request_duration_seconds` (p95), `leads_created_total`, `activities_logged_total`, `db_query_duration_seconds`, `db_pool_connections`, `redis_command_duration_seconds`, default Node.js metrics
- **Health**: `/health/live` (process) and `/health/ready` (PostgreSQL + Redis); both public
- **Trace (optional extension)**: OpenTelemetry + Jaeger. Look at metrics first to spot something abnormal (high p95, error rate), then use the logs and the `traceId` of a slow request to open its trace and see which step is slow
- Review the key requests (`POST /leads`, `POST /leads/:id/activities`, `GET /leads/:id`, `GET /leads`) weekly through metrics

## 10. Scalability & Reliability

- API is stateless; scale horizontally behind a load balancer
- PostgreSQL primary + standby replica for failover; a replica is not a backup, so keep scheduled backups and PITR
- Redis runs as a single instance; if it is down, reads fall back to PostgreSQL (log a warning) and `/health/ready` returns `503`
- Rate limiting on `/auth/login` and `POST /leads`; request timeout and graceful shutdown enabled
- Peaks of ingest traffic can later move behind a queue (`202 Accepted`)

## 11. Testing Strategy

- **Test Location**: unit tests next to source in `tests/` ([name].spec.ts); e2e in `test/`
- **Test Structure**: describe → it (Arrange-Act-Assert)
- **Frameworks**: Jest + Supertest (+ `@nestjs/testing`)
- **Coverage Targets**:
  - Services: 80%+
  - Controllers: 70%+
  - Repositories/data access: 60%
- **Mocking**:
  - Redis and ConfigService in unit tests
  - External services (none today)
- **Integration/e2e** (`.env.test`, database `leads_test`): real PostgreSQL and Redis; cover idempotent ingest, concurrent activities, optimistic-lock conflict (`409`), cache hit/miss, token revocation (logout, deactivate, refresh reuse), role matrix (ADMIN `403` on leads, SALER scope, `assigneeId` `403`) and tenant isolation
