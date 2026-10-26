# Sales Lead Management API Specification

## 1. Overview

- Base URL: `/api/v1`
- API versioning: URL path (`/api/v1/`, `/api/v2/`)
- Content-Type: `application/json`
- Character encoding: UTF-8
- Interactive docs: Swagger UI at `/docs` (dev/staging), the only client for this project
- Identifiers: UUID v4 strings; timestamps: ISO 8601 UTC

## 2. Authentication & Authorization

Two kinds of callers:

| Caller                       | Mechanism                                                | Used by                                    |
| ---------------------------- | -------------------------------------------------------- | ------------------------------------------ |
| ADMIN / MANAGER / SALER      | JWT Bearer token: `Authorization: Bearer <access_token>` | `/leads*`, `/users*`, `/auth/logout`       |
| Dealership website           | API key: `X-API-Key: <key>` (maps to one dealership)     | `POST /leads` only                         |

- Access token: 15 minutes. Refresh token: 7 days, returned in the response body (Swagger-friendly), rotated on every refresh, stored as a SHA-256 hash in the `refresh_tokens` table (one row per device)
- Revocation: access tokens are checked against a Redis blacklist on every request (`bl:jti:{jti}`, `bl:user:{userId}`)
- Public routes: `/auth/login`, `/auth/refresh`, `/health/*`, `/docs`
- There is no public registration. The first ADMIN comes from the seed script; ADMIN creates MANAGER and SALER accounts with `POST /users`
- `dealership_id` always comes from the token or the API key, never from request input

### Roles

| Role    | Scope                                                                                                                  |
| ------- | ---------------------------------------------------------------------------------------------------------------------- |
| ADMIN   | System level (`dealership_id` is null). Manages accounts. **No access to lead endpoints** (403)                         |
| MANAGER | One dealership (a dealership can have many managers). Sees and assigns all leads of the dealership; may change status and assignee |
| SALER   | One dealership. Sees only leads assigned to them; may change the status of their own leads; cannot set `assigneeId` (403) |

A lead outside the caller's scope (other dealership, or not assigned to the SALER) returns **404**, not 403, so its existence is not revealed.

### Permission matrix

| Endpoint                         | Caller              | ADMIN | MANAGER       | SALER            |
| -------------------------------- | ------------------- | :---: | :-----------: | :--------------: |
| POST /auth/login, /auth/refresh  | public              |  ✅   |      ✅       |       ✅         |
| POST /auth/logout                | JWT                 |  ✅   |      ✅       |       ✅         |
| POST /users                      | JWT                 |  ✅   |      ❌ 403   |       ❌ 403     |
| GET /users                       | JWT                 |  ✅   | ✅ own dealership | ❌ 403       |
| PATCH /users/{id}/deactivate     | JWT                 |  ✅   |      ❌ 403   |       ❌ 403     |
| POST /leads                      | API key             |  n/a  |      n/a      |       n/a        |
| GET /leads                       | JWT                 | ❌ 403 | all of dealership | assigned only |
| GET /leads/{id}                  | JWT                 | ❌ 403 | own dealership | assigned only   |
| PATCH /leads/{id}                | JWT                 | ❌ 403 | status + assignee | status only (own leads) |
| POST /leads/{id}/activities      | JWT                 | ❌ 403 | own dealership | assigned only   |
| GET /leads/{id}/activities       | JWT                 | ❌ 403 | own dealership | assigned only   |

## 3. Request Conventions

### Pagination:

- **Lead inbox: cursor (keyset)**: `limit` (default 20, 1-100) and `cursor` (opaque string from `nextCursor`; omit for the first page)
- **Activities and users list: offset**: `offset` (default 0, min 0) and `limit` (default 20, 1-100). The server fetches `limit + 1` rows to compute `hasMore`
- Example: `GET /leads?status=NEW&limit=20&cursor=eyJjcmVhdGVkQXQiOi4uLn0`

### Sorting:

- `GET /leads`: fixed, newest first (`created_at DESC, id DESC`)
- `GET /leads/{id}/activities`: `order=asc|desc` (default `desc`, by `occurred_at` then `id`)

### Filtering (`GET /leads`):

| Query        | Type / rule                                                               |
| ------------ | ------------------------------------------------------------------------- |
| `status`     | `NEW`, `CONTACTED`, `QUALIFIED`, `WON`, `LOST`                            |
| `source`     | string, exact match                                                       |
| `q`          | 1-100 chars, search by customer name or email (case-insensitive)          |
| `assigneeId` | UUID, **MANAGER only** (a SALER sending it gets 403)                      |

### Request headers:

- `Idempotency-Key`: optional UUID v4, only for `POST /leads`; generate one **per form submission** and reuse it on retries
- `X-Request-Id`: optional; generated by the server if missing, always returned in the response header

### Request body:

- JSON, camelCase field names
- Validated by DTOs with class-validator; unknown fields are rejected (`whitelist` + `forbidNonWhitelisted`); violations return `400 VALIDATION_ERROR` with an `errors[]` list

## 4. Response Format

### Success (single item): the resource itself

```json
{ "id": "6f1c2c1e-0b7a-4a40-9d8e-1a2b3c4d5e6f", "status": "NEW", "createdAt": "2026-10-24T08:30:00Z" }
```

### Success (cursor list, `GET /leads`):

```json
{ "items": [ ... ], "nextCursor": "eyJjcmVhdGVkQXQiOi4uLn0" }
```

`nextCursor` is `null` on the last page.

### Success (offset list, activities and users):

```json
{ "items": [ ... ], "offset": 0, "limit": 20, "hasMore": true }
```

### Error (RFC 7807 `application/problem+json`):

```json
{
  "type": "https://api.example.com/problems/lead-not-found",
  "title": "Lead not found",
  "status": 404,
  "code": "LEAD_NOT_FOUND",
  "detail": "Lead 6f1c2c1e-... does not exist",
  "traceId": "a1b2c3d4",
  "errors": [ { "field": "email", "message": "email must be an email" } ]
}
```

`errors[]` appears only for validation errors. Never expose stack traces or SQL errors.

## 5. Error Codes

Codes are named in UPPER_SNAKE_CASE and returned in the `code` field.

| HTTP | Code                       | Description                                                         |
| ---- | -------------------------- | ------------------------------------------------------------------- |
| 400  | VALIDATION_ERROR           | Invalid body, query or path parameter                               |
| 401  | INVALID_CREDENTIALS        | Wrong email/password, or account deactivated                        |
| 401  | UNAUTHENTICATED            | Missing, invalid, expired or revoked token / API key                |
| 403  | FORBIDDEN                  | Valid caller but the role is not allowed (including SALER sending `assigneeId`) |
| 404  | LEAD_NOT_FOUND             | Lead does not exist or is outside the caller's scope                |
| 404  | USER_NOT_FOUND             | User does not exist                                                 |
| 404  | DEALERSHIP_NOT_FOUND       | `dealershipId` in `POST /users` does not exist                      |
| 409  | LEAD_VERSION_CONFLICT      | `version` mismatch: someone else updated the lead                   |
| 409  | EMAIL_ALREADY_EXISTS       | Email is already used by another account                            |
| 422  | INVALID_STATUS_TRANSITION  | Status change not allowed by the lifecycle                          |
| 422  | INVALID_ASSIGNEE           | Assignee is not an active SALER of this dealership                  |
| 429  | RATE_LIMITED               | Too many requests                                                   |
| 500  | INTERNAL_ERROR             | Unexpected server error                                             |
| 503  | SERVICE_UNAVAILABLE        | Dependency unavailable (PostgreSQL / Redis), used by `/health/ready` |

### HTTP status usage:

- 200: Success (GET, PATCH, login, refresh, idempotent replay of `POST /leads`)
- 201: Created (new lead, user, activity)
- 204: No content (logout, deactivate)
- 400: Validation error
- 401: Unauthenticated
- 403: Forbidden
- 404: Not found (also for out-of-scope leads)
- 409: Conflict
- 422: Business rule violation
- 429: Rate limit exceeded
- 500: Internal error
- 503: Not ready

### Lead status lifecycle:

`NEW → CONTACTED → QUALIFIED → WON | LOST`. The first activity moves `NEW → CONTACTED` automatically. `WON` and `LOST` are reachable only from `QUALIFIED`. Any other change returns `422 INVALID_STATUS_TRANSITION`.

## 6. Endpoints by Feature

### Auth Feature:

| Method | Path          | Description                                      | Auth          |
| ------ | ------------- | ------------------------------------------------ | ------------- |
| POST   | /auth/login   | Login, get access + refresh token                | Public        |
| POST   | /auth/refresh | Exchange a refresh token for a new pair (rotation) | Public (refresh token in body) |
| POST   | /auth/logout  | Revoke this device's refresh token, blacklist the access token | JWT |

### User Feature:

| Method | Path                  | Description                                          | Auth                  |
| ------ | --------------------- | ---------------------------------------------------- | --------------------- |
| POST   | /users                | Create a MANAGER or SALER account                    | JWT (ADMIN)           |
| GET    | /users                | List users (MANAGER sees own dealership; used to pick an assignee) | JWT (ADMIN, MANAGER) |
| PATCH  | /users/{id}/deactivate | Disable an employee and revoke all their tokens     | JWT (ADMIN)           |

### Lead Feature:

| Method | Path                  | Description                                                   | Auth                  |
| ------ | --------------------- | ------------------------------------------------------------- | --------------------- |
| POST   | /leads                | Website submits a new lead (`Idempotency-Key`)                | API key               |
| GET    | /leads                | Lead inbox (filterable, cursor-paginated)                     | JWT (MANAGER, SALER)  |
| GET    | /leads/{id}           | Lead detail (lead only, no activities)                        | JWT (MANAGER, SALER)  |
| PATCH  | /leads/{id}           | Update status and/or assignee (requires `version`)            | JWT (MANAGER, SALER)  |
| POST   | /leads/{id}/activities | Log a follow-up activity                                     | JWT (MANAGER, SALER)  |
| GET    | /leads/{id}/activities | List activities (offset-paginated)                           | JWT (MANAGER, SALER)  |

The lead detail screen calls two APIs: `GET /leads/{id}` and `GET /leads/{id}/activities` (defaults).

### Health & Ops Feature:

| Method | Path          | Description                                   | Auth             |
| ------ | ------------- | --------------------------------------------- | ---------------- |
| GET    | /health/live  | Liveness: the process is up                   | Public           |
| GET    | /health/ready | Readiness: PostgreSQL **and** Redis reachable (503 otherwise) | Public |
| GET    | /metrics      | Prometheus metrics                            | Internal network |
| GET    | /docs         | Swagger UI                                    | Public (dev/staging) |

## 7. Endpoint Details

### POST /auth/login

| Field    | Type   | Required | Validation                                  |
| -------- | ------ | :------: | ------------------------------------------- |
| email    | string | yes      | valid email, max 255, lowercased            |
| password | string | yes      | max 72                                      |

```
Response (200):
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
  "expiresIn": 900,
  "user": { "id": "...", "fullName": "Tran Thi B", "email": "b@dealer.com", "role": "SALER", "dealershipId": "..." }
}

Errors: 400 VALIDATION_ERROR, 401 INVALID_CREDENTIALS, 429 RATE_LIMITED
```

### POST /auth/refresh

Body: `{ "refreshToken": "<token>" }` (required string). Returns the same shape as login (new pair). The old token is revoked. Presenting an already revoked token revokes **all** tokens of that user.

Errors: 400 VALIDATION_ERROR, 401 UNAUTHENTICATED, 429 RATE_LIMITED.

### POST /auth/logout

Header `Authorization: Bearer <access_token>`, body `{ "refreshToken": "<token>" }`. Revokes that device's refresh token and blacklists the access token. Response `204`.

Errors: 401 UNAUTHENTICATED.

### POST /users (ADMIN)

| Field        | Type   | Required | Validation                                         |
| ------------ | ------ | :------: | -------------------------------------------------- |
| fullName     | string | yes      | 1-100 chars                                        |
| email        | string | yes      | valid email, max 255, unique                       |
| password     | string | yes      | 8-72 chars, at least one letter and one digit      |
| role         | enum   | yes      | `MANAGER` or `SALER`                               |
| dealershipId | uuid   | yes      | must exist                                         |

```
Response (201): { "id": "...", "fullName": "...", "email": "...", "role": "SALER", "dealershipId": "...", "isActive": true, "createdAt": "..." }

Errors: 400 VALIDATION_ERROR, 401 UNAUTHENTICATED, 403 FORBIDDEN, 404 DEALERSHIP_NOT_FOUND, 409 EMAIL_ALREADY_EXISTS
```

### GET /users (ADMIN, MANAGER)

Query: `role` (`MANAGER|SALER`, optional), `dealershipId` (ADMIN only; a MANAGER is always scoped to their own dealership), `offset`, `limit`.

```
Response (200): { "items": [ { "id": "...", "fullName": "...", "email": "...", "role": "SALER", "dealershipId": "...", "isActive": true } ], "offset": 0, "limit": 20, "hasMore": false }
```

### PATCH /users/{id}/deactivate (ADMIN)

Effect: `is_active = false`, `deactivated_at` set, all refresh tokens revoked, `bl:user:{userId}` written so existing access tokens are rejected. Response `204`.

Errors: 401 UNAUTHENTICATED, 403 FORBIDDEN, 404 USER_NOT_FOUND.

### POST /leads (API key)

Headers: `X-API-Key: <website key>`, `Idempotency-Key: <uuid v4>` (optional).

| Field           | Type   | Required | Validation                                              |
| --------------- | ------ | :------: | ------------------------------------------------------- |
| fullName        | string | yes      | 1-100 chars                                             |
| email           | string | one of email/phone | valid email, max 255                          |
| phone           | string | one of email/phone | 8-15 digits, optional leading `+`             |
| productCategory | string | no       | max 50                                                  |
| productName     | string | no       | max 100                                                 |
| message         | string | no       | max 2000                                                |
| source          | string | no       | max 50, default `WEBSITE`                               |

```
Response (201, new lead) / (200, same Idempotency-Key seen before):
{ "id": "6f1c2c1e-...", "status": "NEW", "createdAt": "2026-10-24T08:30:00Z" }

Errors: 400 VALIDATION_ERROR, 401 UNAUTHENTICATED, 429 RATE_LIMITED
```

No `Idempotency-Key` means a new lead is always created. There is no dedup by name or email.

### GET /leads (MANAGER, SALER)

MANAGER sees every lead of the dealership; SALER only leads where `assignee_id` is themselves. Query: `status`, `source`, `q`, `assigneeId` (MANAGER only), `limit`, `cursor`.

```
Response (200):
{
  "items": [
    { "id": "6f1c2c1e-...", "fullName": "Nguyen Van A", "email": "a@example.com", "phone": "0901234567",
      "productCategory": "vehicle", "productName": "Toyota Camry", "source": "WEBSITE",
      "status": "NEW", "assigneeId": null, "createdAt": "2026-10-24T08:30:00Z" }
  ],
  "nextCursor": "eyJjcmVhdGVkQXQiOi4uLn0"
}

Errors: 400 VALIDATION_ERROR, 401 UNAUTHENTICATED, 403 FORBIDDEN, 429 RATE_LIMITED
```

The list omits `message` to keep the payload small.

### GET /leads/{id} (MANAGER, SALER)

```
Response (200):
{ "id": "...", "dealershipId": "...", "assigneeId": "...", "fullName": "...", "email": "...", "phone": "...",
  "productCategory": "...", "productName": "...", "message": "...", "source": "WEBSITE",
  "status": "CONTACTED", "version": 3, "createdAt": "...", "updatedAt": "..." }

Errors: 400 VALIDATION_ERROR (id not a UUID), 401 UNAUTHENTICATED, 403 FORBIDDEN, 404 LEAD_NOT_FOUND
```

Served from Redis (`lead:{dealershipId}:{leadId}`) when cached.

### PATCH /leads/{id} (MANAGER, SALER)

| Field      | Type    | Required | Validation / rule                                                              |
| ---------- | ------- | :------: | ------------------------------------------------------------------------------ |
| version    | integer | yes      | current version from `GET /leads/{id}`                                         |
| status     | enum    | one of status/assigneeId | must follow the lifecycle                              |
| assigneeId | uuid    | one of status/assigneeId | **MANAGER only**; must be an active SALER of the same dealership; `null` is not allowed |

```
Response (200): the updated lead (same shape as GET /leads/{id}, with the new version)

Errors: 400 VALIDATION_ERROR, 401 UNAUTHENTICATED, 403 FORBIDDEN (ADMIN, or SALER sending assigneeId),
        404 LEAD_NOT_FOUND, 409 LEAD_VERSION_CONFLICT, 422 INVALID_STATUS_TRANSITION, 422 INVALID_ASSIGNEE
```

Optimistic lock: `UPDATE ... SET ..., version = version + 1 WHERE id AND dealership_id AND version`. When 0 rows are affected the lead is re-checked: missing → 404, otherwise → 409 with the message "The lead was updated by someone else. Please reload the page to get the latest status and update again if needed."

### POST /leads/{id}/activities (MANAGER, SALER)

| Field      | Type     | Required | Validation                                                   |
| ---------- | -------- | :------: | ------------------------------------------------------------ |
| type       | enum     | yes      | `CALL`, `EMAIL`, `SMS`, `MEETING`, `NOTE`                    |
| note       | string   | when `type = NOTE` | max 2000                                           |
| occurredAt | datetime | no       | ISO 8601, not in the future, default now                     |

```
Response (201):
{ "id": "...", "leadId": "...", "type": "CALL", "note": "...", "occurredAt": "...", "createdBy": "<userId>", "createdAt": "..." }

Errors: 400 VALIDATION_ERROR, 401 UNAUTHENTICATED, 403 FORBIDDEN, 404 LEAD_NOT_FOUND
```

Side effect (same transaction): `INSERT activity` + `UPDATE leads SET status='CONTACTED', version=version+1 WHERE id=? AND status='NEW'`. Redis cache is rebuilt after COMMIT.

### GET /leads/{id}/activities (MANAGER, SALER)

Query: `offset` (default 0), `limit` (default 20, 1-100), `order` (`asc|desc`, default `desc`).

```
Response (200):
{
  "items": [
    { "id": "...", "type": "CALL", "note": "Left voicemail", "occurredAt": "2026-10-24T09:00:00Z",
      "createdBy": { "id": "...", "fullName": "Tran Thi B" }, "createdAt": "..." }
  ],
  "offset": 0, "limit": 20, "hasMore": false
}

Errors: 400 VALIDATION_ERROR, 401 UNAUTHENTICATED, 403 FORBIDDEN, 404 LEAD_NOT_FOUND
```

Only the default page (`offset=0, limit=20, order=desc`) is cached in Redis.

### GET /health/live, GET /health/ready

`live` always returns `200` while the process runs. `ready` returns `200` when PostgreSQL and Redis answer, otherwise `503 SERVICE_UNAVAILABLE`.

## 8. NestJS-Specific Additions

- Use `@ApiTags`, `@ApiOperation`, `@ApiResponse`, `@ApiBearerAuth`, `@ApiSecurity('api-key')` for Swagger documentation
- DTOs with class-validator decorators; `ValidationPipe` global (`whitelist`, `forbidNonWhitelisted`, `transform`)
- Use `@Query()` for pagination/filter params, `@Param('id', ParseUUIDPipe)` for ids, `@Body()` for the body
- Custom `@CurrentUser()` decorator for the authenticated user (carries `userId`, `dealershipId`, `role`)
- `@Roles(Role.ADMIN)`, `@Roles(Role.MANAGER, Role.SALER)` for role checks (`RolesGuard`), `@Public()` for routes outside `JwtAuthGuard`
- `ThrottlerGuard` (Redis storage): stricter limits on `/auth/login` and `POST /leads`; limits come from config
- Global `HttpExceptionFilter` converts every error to RFC 7807 and adds `traceId`
