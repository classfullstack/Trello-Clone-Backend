# Trello Clone — Backend

Express 4 REST API + Socket.io realtime + Prisma/PostgreSQL. Feature-sliced modules under `src/modules/*` (`routes → controller → service`).

> Run alongside `Trello-Clone-Frontend` and `Trello-Clone-Infra` as sibling repos — see the `Trello-Clone-Infra` repo's README for the full Docker Compose stack.

## Stack

Node.js 20 · Express 4 · Prisma 5 + PostgreSQL 16 · Redis 7 (ioredis) · Socket.io 4 · BullMQ · Zod · JWT + bcryptjs · MinIO (S3) · pino + prom-client + OpenTelemetry.

## Quick start (standalone)

```bash
cp .env.example .env        # fill DATABASE_URL, REDIS_URL, JWT_SECRET, MINIO_*
npm install
npx prisma migrate deploy
npm run seed                 # seeds RBAC roles/permissions (+ super admin)
npm run dev                  # http://localhost:4000, --watch reload
```

Requires a running PostgreSQL, Redis, and MinIO — normally provided by `Trello-Clone-Infra`'s `make dev`.

## Scripts

| Script | Purpose |
|---|---|
| `npm run dev` | dev server, `node --watch` + OTel preload |
| `npm start` | production start |
| `npm run seed` | seed RBAC roles/permissions + super admin (`SEED_SUPER_ADMIN`) |
| `npm run prisma:migrate` | `prisma migrate deploy` |
| `npm run prisma:generate` | regenerate Prisma client |

## Folder structure

```
src/
├── config/        env (Zod-validated), db (Prisma), redis, minio clients
├── middleware/     authenticate (JWT), authorize (RBAC + audit), errorHandler, sanitize
├── modules/        21 feature modules — routes/controller/service/schema
├── realtime/       Socket.io — rooms user:<id>, board:<id>
├── queues/         BullMQ — email, reminders, backup (workers gated by ENABLE_WORKERS)
├── observability/  pino logger, prom-client metrics, OpenTelemetry tracing
├── lib/            AppError, fractional-index positioning, mailer, email templates
└── db/             seed.js, demo-seed.js
```

## Modules

`auth`, `users`, `workspaces`, `boards`, `lists`, `cards`, `comments`, `labels`, `checklists`, `customFields`, `reactions`, `attachments`, `notifications`, `activity`, `rbac`, `search`, `admin`, `me`, `landing`, `backup`, `zalo`.

## Auth model

Access token: JWT (HS256, 15m default), claims `user_id`/`token_version`/`jti`. Refresh token: opaque, hashed at rest, httpOnly cookie, rotation with reuse detection (`RefreshToken.used`). Logout blacklists the access token's `jti` in Redis until natural expiry.

## RBAC

Three tiers via one `UserRole` table (`tenantId` nullable): system roles (`super_admin`/`admin`/`support`/`user`), workspace roles (`ws_owner`>`ws_admin`>`ws_member`>`ws_guest`), board-level access derived from workspace role (MVP rule). `authorize(permission)` middleware checks a Redis-cached (300s) permission set; sensitive actions are written to `AccessAudit`.

## Key env vars

`DATABASE_URL`, `REDIS_URL`, `JWT_SECRET` (required) · `PORT` (4000) · `ACCESS_TOKEN_TTL` (15m) · `REFRESH_TOKEN_TTL_DAYS` (7) · `COOKIE_SECURE` (set `true` behind HTTPS) · `MINIO_*` · `SMTP_*` · `ENABLE_WORKERS` · `DEEPSEEK_API_KEY`/`DEEPSEEK_MODEL` · `ZALO_BOT_TOKEN`/`ZALO_CHAT_ID`/`ZALO_WEBHOOK_SECRET`. Full list + defaults in `src/config/env.js` / `.env.example`.

## Observability

`GET /health` (DB+Redis check), `GET /metrics` (Prometheus, unauthenticated — do not expose publicly). Structured JSON logs via pino, trace-correlated when `OTEL_EXPORTER_OTLP_ENDPOINT` is set.

## Testing

No automated test suite yet.
