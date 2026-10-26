# TODO – Sales Lead Management (NestJS + TypeORM + PostgreSQL + Redis)

## Giai đoạn 0 – Khởi tạo dự án

### 0.1 Môi trường

- [x] Kiểm tra: `node -v`, `psql --version`, `redis-cli ping` (trả `PONG`), `nest --version`, `git --version`
- [x] Local chạy PostgreSQL và Redis cài sẵn (không dùng Docker); Docker chỉ dùng cho Prometheus/Grafana và đóng gói ở giai đoạn 6 và 8

### 0.2 Tạo project

```bash
nest new .
```

- [ ] `git log --oneline` thấy commit đầu; `npm run start:dev` chạy được

### 0.3 Cài thư viện

```bash
npm i @nestjs/config @nestjs/typeorm typeorm pg
npm i ioredis @nestjs/cache-manager cache-manager keyv @keyv/redis
npm i @nestjs/throttler @nest-lab/throttler-storage-redis
npm i @nestjs/passport @nestjs/jwt passport passport-jwt bcrypt
npm i -D @types/passport-jwt @types/bcrypt
npm i class-validator class-transformer @nestjs/swagger @nestjs/terminus
npm i nestjs-pino pino-http pino-pretty
npm i @willsoto/nestjs-prometheus prom-client
npm i -D supertest @types/supertest
```

- [ ] `npm run build` không lỗi (nếu `bcrypt` không build được trên Node mới thì dùng `bcryptjs` và ghi chú lại)
- [ ] Bật `strict`, cấu hình ESLint + Prettier

### 0.4 Database và Redis

```bash
psql -U postgres -c "CREATE USER leads WITH PASSWORD 'leads';"
psql -U postgres -c "CREATE DATABASE leads OWNER leads;"
psql -U postgres -c "CREATE DATABASE leads_test OWNER leads;"
```

- [ ] E2E dùng database `leads_test` và Redis DB số 1 (`redis://localhost:6379/1`), xóa dữ liệu test sau mỗi lần chạy

### 0.5 Biến môi trường

- [ ] Tạo `.env.example` (commit), `.env`, `.env.test` (không commit, có trong `.gitignore`)

```
NODE_ENV=development
PORT=3000
DATABASE_URL=postgres://leads:leads@localhost:5432/leads
REDIS_URL=redis://localhost:6379
JWT_ACCESS_SECRET=change-me-access
JWT_ACCESS_EXPIRES_IN=15m
JWT_REFRESH_SECRET=change-me-refresh
JWT_REFRESH_EXPIRES_IN=7d
INGEST_API_KEYS=demo-key:<dealershipId>
LEAD_CACHE_TTL_SECONDS=300
THROTTLE_TTL=60
THROTTLE_LIMIT=60
LOG_LEVEL=debug
```

### 0.6 Cấu trúc thư mục và script

- [ ] Tạo thư mục theo `BE-PROJECT-RULES.md` mục 1: `auth/ users/ dealerships/ leads/ health/ metrics/ redis/ config/ database/ common/`
- [ ] Thêm script `package.json`: `typeorm`, `migration:generate`, `migration:run`, `migration:revert`, `seed`
- [ ] Commit: `chore: scaffold project structure`

**Xong khi:**

- `npm run start:dev` chạy, Postgres (`leads`) và Redis chạy, `.env` không bị commit

---

## Giai đoạn 1 – Nền tảng

- [ ] `ConfigModule` global + validate biến môi trường khi khởi động
- [ ] `TypeOrmModule.forRootAsync` (`synchronize: false`) + `src/database/data-source.ts`
- [ ] `RedisModule` global (`ioredis`) + `CacheModule` (Keyv + Redis)
- [ ] `main.ts`: global prefix + URI versioning (`/api/v1`), `ValidationPipe` (`whitelist`, `forbidNonWhitelisted`, `transform`), `enableShutdownHooks()`
- [ ] `common/`: enum `Role` (`ADMIN|MANAGER|SALER`), `LeadStatus`, `ActivityType`; decorator `@Public()`, `@Roles()`, `@CurrentUser()`
- [ ] Logging `nestjs-pino`: JSON, `requestId`, `traceId`, `redact` (`authorization`, password, refresh token, email, phone, message, note); middleware `X-Request-Id`
- [ ] Mức log: `error` = 5xx/4xx bất thường/rate limit/token bị từ chối; `warn` = sự cố nhưng request vẫn chạy; `info` = request bình thường (kể cả 400, 404)
- [ ] `HttpExceptionFilter` trả RFC 7807 (`type, title, status, code, detail, traceId, errors[]`); tạo class exception có mã lỗi theo bảng ở `API_SPEC.md` mục 5 (`LEAD_NOT_FOUND`, `LEAD_VERSION_CONFLICT`, ...)
- [ ] `TimeoutInterceptor`
- [ ] Swagger `/docs` có nút Authorize cho Bearer và `X-API-Key`
- [ ] `HealthModule`: `/health/live`, `/health/ready` (Postgres + Redis, lỗi trả `503 SERVICE_UNAVAILABLE`)

**Xong khi:**

- `/docs` mở được, `/health/ready` trả `200`
- tắt Redis thì trả `503`
- log ra JSON
- validation sai trả RFC 7807 `400 VALIDATION_ERROR`

---

## Giai đoạn 2 – Cơ sở dữ liệu

- [ ] Entity `Dealership`, `User`, `RefreshToken`, `Lead`, `Activity` đúng theo `DATABASE.md` (`@VersionColumn` cho `Lead.version`)
- [ ] Migration đầu tiên (`migration:generate`), kiểm tra bằng tay:
  - [ ] CHECK `users`: ADMIN có `dealership_id` null, role khác thì bắt buộc có
  - [ ] CHECK `leads`: `email IS NOT NULL OR phone IS NOT NULL`
  - [ ] Index leads: `(dealership_id, created_at DESC, id DESC)`, `(dealership_id, assignee_id, created_at DESC, id DESC)`, `(dealership_id, status, created_at DESC)`, `(dealership_id, lower(email))`
  - [ ] **Partial unique index** `(dealership_id, idempotency_key) WHERE idempotency_key IS NOT NULL`
  - [ ] Index activities `(lead_id, occurred_at DESC, id DESC)`; refresh_tokens `UNIQUE(token_hash)` và `(user_id)`
- [ ] `migration:run` trên DB sạch, thử `migration:revert`
- [ ] Seed: 1 dealership, 1 ADMIN, 1 MANAGER, 2 SALER (bcrypt), vài lead (có lead chưa gán, lead đã gán cho từng SALER), vài activity

**Xong khi:**

- DB dựng lại từ đầu chỉ bằng `migration:run` + `seed`
- `\d leads` thấy đủ cột, constraint, index

---

## Giai đoạn 3 – Xác thực và phân quyền

- [ ] `UsersModule`: `UsersService` (tìm theo email, theo id; export)
- [ ] `POST /auth/login`: bcrypt, từ chối `is_active = false` (`401 INVALID_CREDENTIALS`), trả `{ accessToken, refreshToken, expiresIn: 900, user }`
- [ ] Lưu refresh token **SHA-256 hash** vào `refresh_tokens` (một dòng mỗi thiết bị, `device_info` từ User-Agent, `expires_at` +7 ngày)
- [ ] `POST /auth/refresh`: rotation (dòng cũ `revoked_at`, tạo dòng mới); dùng lại token đã thu hồi thì thu hồi **toàn bộ** token của user
- [ ] `JwtAuthGuard` toàn cục (kiểm tra blacklist Redis `bl:jti:{jti}` và `bl:user:{userId}`) + `RolesGuard`
- [ ] `POST /auth/logout`: thu hồi refresh token của thiết bị đó + ghi `bl:jti:{jti}` (TTL = thời gian còn lại của access token), trả `204`
- [ ] `POST /users` (ADMIN): validate theo `API_SPEC.md` (password 8–72, có chữ và số; `409 EMAIL_ALREADY_EXISTS`; `404 DEALERSHIP_NOT_FOUND`)
- [ ] `GET /users` (ADMIN, MANAGER): MANAGER chỉ thấy đại lý của mình; phân trang `offset/limit` → `{ items, offset, limit, hasMore }`
- [ ] `PATCH /users/:id/deactivate` (ADMIN): `is_active = false`, `deactivated_at`, thu hồi mọi refresh token, ghi `bl:user:{userId}`, trả `204`
- [ ] Throttler riêng cho `/auth/login` và `/auth/refresh` (`429 RATE_LIMITED`)

**Xong khi:**

- đăng nhập được trong Swagger
- đăng nhập 2 thiết bị độc lập
- logout xong token cũ `401`
- dùng lại refresh token cũ làm mất toàn bộ phiên
- deactivate xong nhân viên không login/refresh được và access token đang dùng bị `401`
- SALER/MANAGER gọi `POST /users` nhận `403`

---

## Giai đoạn 4 – Nhận lead từ website (ingest)

- [ ] `ApiKeyGuard` đọc `X-API-Key`, ánh xạ ra `dealershipId` từ `INGEST_API_KEYS` (sai/thiếu → `401 UNAUTHENTICATED`)
- [ ] `CreateLeadDto`: `fullName` 1–100 bắt buộc; `email`/`phone` ít nhất một (phone 8–15 chữ số, `+` đầu tùy chọn); `productCategory` ≤50; `productName` ≤100; `message` ≤2000; `source` ≤50 mặc định `WEBSITE`
- [ ] `POST /leads`: có `Idempotency-Key` → SELECT `(dealership_id, key)`; có rồi trả `200`, chưa có INSERT trả `201` (`{ id, status, createdAt }`); không có key luôn tạo mới
- [ ] Xử lý đua: bắt unique violation (partial index), đọc lại lead và trả `200`
- [ ] Throttler cho endpoint này

**Xong khi:**

- hai lần cùng key ra một lead
- hai lần không key ra hai lead
- 10 request song song cùng key vẫn một lead
- thiếu cả email lẫn phone trả `400`

---

## Giai đoạn 5 – Lead inbox, chi tiết, cập nhật

- [ ] Hàm dùng chung kiểm tra phạm vi: MANAGER theo `dealership_id`; SALER thêm `assignee_id = userId`; ngoài phạm vi → `404 LEAD_NOT_FOUND`; ADMIN → `403`
- [ ] `GET /leads`: lọc `status`, `source`, `q` (1–100), `assigneeId` (chỉ MANAGER, SALER gửi → `403`); cursor trên `(created_at, id)`, `limit` 1–100 mặc định 20 → `{ items, nextCursor }` (không có `message`)
- [ ] `GET /leads/:id`: chỉ trả lead (kèm `version`, `dealershipId`, `message`, `updatedAt`); đọc cache `lead:{dealer}:{id}` trước, miss thì đọc DB rồi SET TTL ~5 phút; cache lưu `assigneeId` để kiểm tra phạm vi SALER; 404 không cache
- [ ] `PATCH /leads/:id`: body `version` bắt buộc + ít nhất một trong `status`/`assigneeId`
  - [ ] Luồng trạng thái `NEW → CONTACTED → QUALIFIED → WON|LOST` (WON/LOST chỉ từ QUALIFIED), sai → `422 INVALID_STATUS_TRANSITION`
  - [ ] `assigneeId`: chỉ MANAGER (SALER → `403`), phải là SALER active cùng đại lý, sai → `422 INVALID_ASSIGNEE`
  - [ ] `UPDATE ... SET ..., version = version + 1 WHERE id AND dealership_id AND version`; 0 dòng → `404` hoặc `409 LEAD_VERSION_CONFLICT` (thông báo: lead đã được người khác cập nhật, tải lại trang)
  - [ ] Sau thành công: DEL cache lead (và cache activities nếu đổi `assigneeId`)

**Xong khi:**

- MANAGER thấy mọi lead, SALER chỉ thấy lead được gán
- lead đại lý khác trả `404`
- hai `PATCH` cùng `version` thì một cái nhận `409`
- ADMIN nhận `403`

---

## Giai đoạn 6 – Activity

- [ ] `CreateActivityDto`: `type` (`CALL|EMAIL|SMS|MEETING|NOTE`), `note` ≤2000 (bắt buộc khi `NOTE`), `occurredAt` ISO 8601 không ở tương lai (mặc định now)
- [ ] `POST /leads/:id/activities`: kiểm tra phạm vi, rồi một transaction `INSERT activity` + `UPDATE leads SET status='CONTACTED', version=version+1 WHERE id=? AND status='NEW'`; trả `201`
- [ ] Sau `COMMIT`: đọc lại lead + 20 activity mới nhất, SET cả hai key cache; Redis lỗi thì log `warn` và DEL key
- [ ] `GET /leads/:id/activities`: `offset` (mặc định 0), `limit` 1–100 (mặc định 20), `order` (mặc định `desc`); lấy `limit + 1` để tính `hasMore`; trả `createdBy { id, fullName }`
- [ ] Chỉ cache trang mặc định (`offset=0, limit=20, order=desc`) ở `lead:{dealer}:{id}:activities`; trang khác đọc thẳng DB
- [ ] Retry một lần khi gặp `40P01`/`40001`

**Xong khi:**

- thêm activity xong `GET /leads/:id` và `GET /leads/:id/activities` thấy dữ liệu mới ngay
- lead `NEW` chuyển `CONTACTED`
- ghi nhiều activity song song không lỗi

---

## Giai đoạn 7 – Observability

- [ ] `MetricsModule` + `/metrics` (chỉ mạng nội bộ) với metrics mặc định của Node.js
- [ ] `MetricsInterceptor`: `http_requests_total`, `http_request_duration_seconds` (nhãn `route`, `method`, `status`)
- [ ] Metrics nghiệp vụ: `leads_created_total{source}`, `activities_logged_total{type}`
- [ ] Metrics hạ tầng: `db_query_duration_seconds`, `db_pool_connections`, `redis_command_duration_seconds`
- [ ] Log đủ `timestamp, level, msg, requestId, traceId, userId, dealershipId, route, statusCode, durationMs`; kiểm tra không lộ email, phone, message, note, token
- [ ] `docker-compose.yml` thêm `prometheus` (và `grafana`); mở `localhost:9090` thấy target `UP`; dựng dashboard p95 cho 4 request chính (`POST /leads`, `POST /leads/:id/activities`, `GET /leads/:id`, `GET /leads`)
- [ ] (Tùy chọn, làm cuối) OpenTelemetry + Jaeger: xem metrics trước, sau đó dùng `traceId` của request chậm để mở trace

**Xong khi:**

- tạo vài lead qua Swagger rồi `/metrics` thấy `leads_created_total` tăng
- truy vấn p95 trong Prometheus có kết quả

---

## Giai đoạn 8 – Kiểm thử

- [ ] Unit test service: `LeadsService`, `AuthService`, `UsersService` (mock Redis, repository)
- [ ] E2E (Supertest + Postgres + Redis thật, `.env.test`):
  - [ ] Auth: login, refresh rotation, dùng lại token đã thu hồi, logout, deactivate, đăng nhập nhiều thiết bị
  - [ ] Ingest idempotent (có/không key, gọi song song)
  - [ ] Inbox + cursor + bộ lọc; ma trận role (ADMIN `403`, SALER chỉ lead được gán, SALER gửi `assigneeId` `403`)
  - [ ] Cách ly đại lý: user đại lý A không thấy lead của B (`404`)
  - [ ] Cache hit/miss; thêm activity cập nhật cache; đổi `assigneeId` làm mất cache activities
  - [ ] `409` khi sai `version`; `422` khi chuyển trạng thái sai hoặc gán sai người
  - [ ] Nhiều activity đồng thời cho cùng lead không lỗi, không mất dữ liệu
- [ ] `npm run test:cov`: service ≥80%, controller ≥70%

**Xong khi:**

- `npm test` và `npm run test:e2e` đều xanh

---

## Giai đoạn 9 – Hoàn thiện và bàn giao

- [ ] `Dockerfile` (multi-stage) và service `api` trong `docker-compose.yml`
- [ ] GitHub Actions: lint, typecheck, test, build
- [ ] `README.md`: clone → `.env` → khởi động Postgres/Redis → `migration:run` → `seed` → `npm run start:dev`; tài khoản demo (ADMIN, MANAGER, SALER) và link `/docs`
- [ ] Đối chiếu tài liệu thiết kế, `API_SPEC.md`, `DATABASE.md` với code thật; sửa chỗ lệch
- [ ] Ghi lại cách dùng AI (mục 9 tài liệu thiết kế): dùng ở đâu, sửa gì, kiểm chứng thế nào
- [ ] Kiểm tra cuối: clone repo vào thư mục mới, làm theo README, chạy được từ đầu đến cuối

**Xong khi:**

- người khác làm theo README là chạy được hệ thống và gọi thử mọi API trong Swagger
