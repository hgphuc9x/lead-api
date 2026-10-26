## Tech Stack

- Language: TypeScript (strict)
- Framework: NestJS (modular monolith)
- ORM: TypeORM (with @nestjs/typeorm), migrations only (`synchronize: false`)
- Database: PostgreSQL
- Cache / shared state: Redis (token blacklist, rate-limit counters, lead cache) - mandatory
- API docs: Swagger UI (@nestjs/swagger), the only client
- Logging / metrics: nestjs-pino, Prometheus (@willsoto/nestjs-prometheus)
- Tests: Jest + Supertest

## Requirements

### 1. Feature Structure

```
src/
├── auth/           # login, refresh, logout, JWT strategy, guards, refresh_tokens, token blacklist
├── users/          # user entity, create account (ADMIN), list users, deactivate employee
├── dealerships/    # dealership entity
├── leads/          # leads + activities (activity is part of the lead aggregate), ingest
├── health/         # /health/live, /health/ready
├── metrics/        # Prometheus metrics, MetricsInterceptor
├── redis/          # global RedisModule (cache, blacklist client)
├── config/         # typed configuration, env validation
├── database/       # data-source.ts, migrations/, seeds/
└── common/         # decorators, filters, interceptors, pipes, dto, enums, constants
```

Each feature folder follows this structure (example: leads):

```
leads/
├── leads.module.ts
├── leads.controller.ts
├── leads.service.ts
├── dto/
│   ├── create-lead.dto.ts
│   ├── update-lead.dto.ts
│   ├── create-activity.dto.ts
│   └── list-leads-query.dto.ts
├── entities/
│   ├── lead.entity.ts
│   └── activity.entity.ts
├── types/
│   └── leads.types.ts
├── tests/
│   ├── leads.controller.spec.ts
│   └── leads.service.spec.ts
└── CONTEXT.md
```

### 2. Naming Conventions

- Folders: kebab-case (auth, users, leads, common)
- Files: kebab-case (create-lead.dto.ts, lead.entity.ts)
- Classes: PascalCase (LeadsService, CreateLeadDto)
- Functions/Methods: camelCase (findById, addActivity)
- Variables: camelCase (leadId, dealershipId, assigneeId)
- Constants: UPPER_SNAKE_CASE (DEFAULT_PAGE_LIMIT, LEAD_CACHE_TTL_SECONDS)
- Interfaces/Types: PascalCase with prefix I or T (IJwtPayload, TLeadCursor)
- Entities: PascalCase singular (User, Lead, Activity, RefreshToken); tables plural snake_case (leads)
- DB columns snake_case, entity properties camelCase (`@Column({ name: 'dealership_id' })`)
- Enums: PascalCase name, UPPER_SNAKE_CASE values (Role.SALER, LeadStatus.NEW, ActivityType.CALL)
- Error codes: UPPER_SNAKE_CASE names (LEAD_NOT_FOUND), listed in API_SPEC.md

### 3. Feature Rules

- Each folder under src/ is a feature or shared module and must be self-contained.
- No direct imports between feature internal files (use module exports).
- Cross-feature communication via:
  - Module exports/imports through NestJS DI (e.g. LeadsModule imports UsersModule)
  - Shared code in common/, config/, redis/
- Feature dependencies are declared explicitly in module imports.

Feature boundaries for this project:

- auth: authentication only (login, refresh, logout, blacklist, refresh_tokens); depends on users and redis
- users: user data, account creation, listing and deactivation; standalone
- dealerships: dealership entity; standalone
- leads: lead ingest, inbox, detail, status/assignee update, activities; depends on users and redis
- health: probes; depends on typeorm and redis
- metrics: Prometheus registry and interceptor; standalone

Roles:

- ADMIN: system level (no dealership), manages accounts, no lead access (403)
- MANAGER: one dealership, all its leads, may change status and assignee
- SALER: one dealership, only leads assigned to them, may change status but never the assignee (403)
- A lead outside the caller's scope returns 404

### 4. Code Patterns (MUST follow)

**Error handling:**

- Use NestJS built-in exceptions (NotFoundException, ConflictException, ...)
- Custom exceptions extend HttpException and carry a named error code (e.g. LEAD_NOT_FOUND, LEAD_VERSION_CONFLICT)
- Global HttpExceptionFilter in common/filters returns RFC 7807 with `traceId`

**Validation:**

- class-validator decorators in DTOs
- ValidationPipe global: `whitelist`, `forbidNonWhitelisted`, `transform`
- `ParseUUIDPipe` on every `:id` parameter
- Business rules (status transition, assignee, role scope) live in the service, not the DTO

**Logging:**

- Use nestjs-pino (structured JSON), never `console.log`
- Log at service level, not controller; every line carries `requestId`
- Levels: `error` = 5xx, abnormal 4xx, rate limit, rejected tokens; `warn` = incident but the request still works (Redis down → DB fallback, slow request); `info` = normal requests incl. 400/404; `debug` = dev only
- Redact: `authorization`, cookies, passwords, tokens, customer email/phone/message, activity note
- Format: `this.logger.info({ leadId, type }, 'activity created')`

**Response format:**

- Success: the resource itself
- Lead inbox: `{ items, nextCursor }`; activities and users: `{ items, offset, limit, hasMore }`
- Error: RFC 7807 `{ type, title, status, code, detail, traceId, errors? }` (`errors` for validation only)

**TypeORM pattern:**

- Each entity lives in `entities/`; access data via `Repository` / `QueryBuilder` in the service
- Multi-table writes in `dataSource.transaction()`
- Every query filters by `dealership_id` taken from the authenticated user or API key; SALER queries also filter by `assignee_id`
- Lead inbox: cursor pagination on `(created_at, id)`; activities and users list: `LIMIT/OFFSET` fetching `limit + 1` (activities per lead stay small and indexed)
- Optimistic locking: `PATCH /leads/:id` requires `version`; update with `WHERE ... AND version = ?` and `version = version + 1` (QueryBuilder updates do not bump `@VersionColumn` for you); 0 rows → 404 or 409
- Automatic transition: conditional `UPDATE ... WHERE status = 'NEW'` (also increments `version`)
- Hash passwords with bcrypt; hash refresh tokens with SHA-256 before storing them in `refresh_tokens` (one row per device)
- Redis is never the source of truth: PostgreSQL first, cache after COMMIT; on a cache write failure log a warning and `DEL` the key
- Cache keys: `lead:{dealershipId}:{leadId}` (lead only) and `lead:{dealershipId}:{leadId}:activities` (default page only: offset 0, limit 20, desc)

### 5. Anti-patterns (MUST NOT do)

- Import directly from another feature's internal files (use module exports)
- Create circular dependencies between features
- Put business logic in controllers (controllers handle HTTP only)
- Use `synchronize: true` or edit the schema outside a migration
- Query without `dealership_id` or take it from request body/query
- Hardcode configuration values (use ConfigService and environment variables)
- Store plain text passwords or plain refresh tokens
- Put PII (email, phone, message, note) into logs, metrics labels or trace attributes
- Use high-cardinality values (leadId, email) as metric labels
- Update or delete rows in `activities` (append-only)
- Add dedup by name/email: duplicates are prevented **only** by `Idempotency-Key`
- Update a lead without checking `version` on manual edits, or forget `version = version + 1` on any lead UPDATE
- Let ADMIN call lead endpoints, or let a SALER set `assigneeId`
- Cache non-default activity pages or cache a 404

**DO vs DON'T example (controller vs service):**

```typescript
// DO: controller delegates, service holds the logic
@Post(':id/activities')
@Roles(Role.MANAGER, Role.SALER)
add(@CurrentUser() user: IJwtPayload, @Param('id', ParseUUIDPipe) id: string, @Body() dto: CreateActivityDto) {
  return this.leadsService.addActivity(user, id, dto);
}

// DON'T: DB access and rules inside the controller
@Post(':id/activities')
async add(@Param('id') id: string, @Body() dto: any) {
  const lead = await this.leadRepo.findOne({ where: { id } }); // no dealership filter, no validation
  ...
}
```

### 6. Git Workflow

- Branch naming: [type]/[feature]-[short-description]
  - feature/leads-cursor-pagination
  - fix/activity-cache-refresh
  - refactor/auth-token-blacklist
- Commit message: [type]: [description]
  - feat: add idempotent lead ingest
  - fix: refresh lead cache after activity commit
  - refactor: move blacklist logic to auth service
- PR requirements:
  - Linked to issue/task
  - Passes lint, typecheck and all tests
  - Includes a migration when entities change
  - Updates DATABASE.md and API_SPEC.md in the same change as any entity or endpoint change
  - Reviewed by at least 1 team member

### 7. Testing

- Unit tests: same folder as source (`tests/` inside the feature), file name `[name].spec.ts`
- E2E tests: `test/*.e2e-spec.ts` with Supertest against a real PostgreSQL + Redis (`.env.test`, database `leads_test`)
- Test structure: describe → it (Arrange-Act-Assert)
- Coverage requirements:
  - Services: 80%+
  - Controllers: 70%+
  - Repositories/data access: 60%+
- Must-have scenarios: idempotent replay of `POST /leads`, concurrent activity writes, optimistic-lock conflict (`409`), cache hit/miss, revoked token rejected (logout, deactivate, refresh reuse), role matrix (ADMIN `403` on leads, SALER sees only assigned leads, SALER `assigneeId` `403`), cross-dealership access returns 404
- Mock external dependencies (Redis in unit tests, clock, ConfigService)

## NestJS-Specific Additions

- Use @Module, @Controller, @Injectable decorators properly
- Guards: `JwtAuthGuard` (global, `@Public()` opts out), `ApiKeyGuard` (ingest), `RolesGuard`, `ThrottlerGuard`
- Interceptors: `MetricsInterceptor`, `TimeoutInterceptor`
- Pipes: global `ValidationPipe`, `ParseUUIDPipe`
- Custom decorators: `@CurrentUser()`, `@Roles()`, `@Public()`
- Entities use `@Entity`, `@Column`, `@Index`, `@VersionColumn`
- Enable shutdown hooks (`app.enableShutdownHooks()`) so DB and Redis connections close cleanly
