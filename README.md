# tracking-service

PostgreSQL/Supabase-compatible clone of `trackingdetail` (`server/trackingdetail`) — a small microservice that lets customers verify their email via OTP and then look up their orders, tracking, and submit shipping-address change requests, without needing full account auth.

The **only** structural change vs. the original: the database layer goes through `./db-adapter.js`, a drop-in `mysql2/promise`-compatible shim backed by `pg` (copied from `order-service`), instead of `mysql2/promise` directly. All route logic, OTP flow, and response shapes are unchanged. Email goes through the shared `../emailService.js` (`Dev/emailService.js`, the same module `admin-service` uses its own copy of), which already has the `tracking_otp` template.

## Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js >= 18 (ESM) |
| Framework | Express 4 |
| Database | PostgreSQL via `pg`, through `db-adapter.js` (mysql2-compatible shim) — same Supabase project as `admin-service` |
| Auth | Email OTP -> short-lived JWT (`jsonwebtoken`) |
| Email | `nodemailer` / Resend / Mailjet via `emailService.js` |

## Getting started

```bash
npm install
npm run dev    # nodemon, auto-restart
npm start      # plain node
```

Copy `.env.example` to `.env` and fill in real values (a working local `.env` pointing at the shared Supabase DB is already present).

The server starts on port `5002` by default (`TRACKING_SERVICE_PORT`).

## Routes

| Method & path | Auth | Purpose |
|---|---|---|
| `GET /health` | No | DB liveness probe. |
| `POST /api/trackingdetail/request-otp` | No | Emails a 6-digit OTP if the address has at least one order. Always returns `{ ok: true }` (no email enumeration). |
| `POST /api/trackingdetail/verify-otp` | No | Verifies the OTP, returns a short-lived JWT (`TRACKING_ACCESS_TOKEN_TTL_SECONDS`, default 15 min). |
| `GET /api/trackingdetail/orders` | Bearer token or `x-api-key` | Orders + items + payments for the verified email. |
| `POST /api/trackingdetail/orders` | Bearer token or `x-api-key` | Same, POST body form. |
| `POST /api/trackingdetail/orders-by-email` | No | Same lookup by raw email, no OTP — for already-authenticated callers. |
| `GET /api/trackingdetail/tracking/:trackingNumber` | Bearer token or `x-api-key` | Fetches carrier tracking events from an external provider (needs `TRACKING_EVENTS_API_BASE_URL`; returns a clean error if unconfigured). |
| `GET /api/trackingdetail/address-change-requests` | Bearer token or `x-api-key` | Lists the verified customer's address-change requests. |
| `POST /api/trackingdetail/order/:orderNumber/address-change-request` | Bearer token or `x-api-key` | Submits a shipping-address change request (rejects if one is already pending for that order). |

`orders`, `order_items`, `payments`, and `order_address_change_requests` are shared tables in the same Supabase database that `admin-service` owns/manages — this service only reads them (plus writes new `order_address_change_requests` rows) and never redefines their schema beyond an idempotent `CREATE TABLE IF NOT EXISTS`.

## Notes on the MySQL -> Postgres port

- `db-adapter.js` auto-translates `?` placeholders, MySQL DDL (`AUTO_INCREMENT`, `DATETIME`, inline `INDEX`/`KEY`, `ENGINE=...`), `DATE_ADD`, and `IF(...)`. See its header comment for the full list.
- Postgres does not support `UPDATE ... LIMIT`, which the original MySQL code used twice (`UPDATE tracking_otps ... WHERE id = ? LIMIT 1`). Both were harmless in MySQL (already unique by `id`) and the `LIMIT 1` was dropped when porting — behavior is identical since `id` is already a primary key.
- Inline `INDEX`/`KEY` clauses in `CREATE TABLE` are stripped by the adapter (Postgres has no inline index syntax) rather than converted to `CREATE INDEX`. Fine for these low-traffic tables; add explicit `CREATE INDEX IF NOT EXISTS` statements later if query volume grows.
