# tracking-service

A small microservice that lets customers verify their email via OTP and then look up their orders, tracking, and submit shipping-address change requests, without needing full account auth.


## Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js >= 18 (ESM) |
| Framework | Express 4 |
| Database | PostgreSQL via `pg`, through `db-adapter.js` (mysql2-compatible shim)|
| Auth | Email OTP -> short-lived JWT (`jsonwebtoken`) |
| Email | `nodemailer` / Resend / Mailjet via `emailService.js` |

## Getting started

```bash
npm install
npm run dev    # nodemon, auto-restart
npm start      # plain node
```

Copy `.env.example` to `.env` and fill in real values.

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

