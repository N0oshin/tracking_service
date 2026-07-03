# tracking-service — Function Reference



## Configuration helpers

### `env(name, fallback = '')`
Reads `process.env[name]`. Returns `fallback` if the variable is unset, `null`, or blank after trimming; otherwise returns it as a string.

**Used by**: every config constant at module load (`PORT`, `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASS`, `DB_NAME`, `corsOrigin`, `API_KEY`, `JWT_SECRET`, `OTP_TTL_SECONDS`, `OTP_MAX_ATTEMPTS`, `ACCESS_TOKEN_TTL_SECONDS`), and inside `fetchTrackingEventsExternal` (`TRACKING_EVENTS_API_BASE_URL`, `TRACKING_EVENTS_API_KEY`, `TRACKING_EVENTS_DEFAULT_CARRIER`). Not tied to a specific endpoint — it's read once at startup / on each tracking lookup.

### `envInt(name, fallback)`
Same as `env`, but coerces the result to a `Number` via `env()` and returns `fallback` if the parsed value isn't finite.

**Used by**: `PORT`, `DB_PORT`, and the three OTP-related tunables (`OTP_TTL_SECONDS`, `OTP_MAX_ATTEMPTS`, `ACCESS_TOKEN_TTL_SECONDS`) at module load.

---

## Date/time helpers

### `nowMysqlDatetime()`
Returns the current time as a `YYYY-MM-DD HH:MM:SS` string (local server time) — the format the mysql2-derived query layer expects for `DATETIME` columns.

**Used by**:
- `POST /api/trackingdetail/request-otp` — stamps `tracking_otps.created_at`.
- `POST /api/trackingdetail/verify-otp` — stamps `tracking_otps.used_at` when an OTP is successfully consumed.
- `POST /api/trackingdetail/order/:orderNumber/address-change-request` — stamps `order_address_change_requests.created_at` and `.updated_at`.

### `addSecondsMysql(sec)`
Returns a `YYYY-MM-DD HH:MM:SS` string `sec` seconds in the future — used to compute OTP expiry.

**Used by**: `POST /api/trackingdetail/request-otp` — sets `tracking_otps.expires_at` to `now + OTP_TTL_SECONDS`.

---

## Schema bootstrap (startup only)

### `ensureTrackingOtpsTable()`
Idempotently creates the `tracking_otps` table (`CREATE TABLE IF NOT EXISTS`) with columns `id, email, otp_hash, otp_salt, expires_at, used_at, attempts, created_at`. Errors are logged, not thrown — a failure here doesn't stop the server from starting (though OTP routes will then fail at query time).

**Used by**: no endpoint directly — called once from the startup IIFE at the bottom of the file, before the server accepts meaningful OTP traffic.

### `ensureOrderAddressChangeRequestsTable()`
Idempotently creates `order_address_change_requests` (`id, order_id, order_number, customer_email, current_shipping_json, requested_shipping_json, status, admin_id, admin_note, decided_at, created_at, updated_at`). This table is **shared with `admin-service`**, which manages the authoritative schema against the same Supabase database — here it's a safety no-op if the table already exists with matching columns.

**Used by**: no endpoint directly — called once from the startup IIFE.

---

## Auth / access-control

### `getBearerToken(req)`
Extracts the token from an `Authorization: Bearer <token>` header via regex; returns `''` if absent or malformed.

**Used by**: `requireOtpAccess` (below).

### `requireOtpAccess(req, res, next)`
Express middleware gating the customer-facing data routes. Logic:
1. If `TRACKING_API_KEY` is configured server-side and the request's `x-api-key` header matches it, allow through immediately (trusted server-to-server bypass).
2. Otherwise require a `Bearer` JWT (via `getBearerToken`), verify it with `JWT_SECRET`, extract and re-validate the embedded `email` claim with `normalizeEmail`, and attach it to `req.otpEmail`.
3. Rejects with `401` if no token, an invalid/expired token, or a token with a bad email claim; `500` if `JWT_SECRET` isn't configured server-side.

**Used by** (as middleware, applied directly in the route registration):
- `GET /api/trackingdetail/tracking/:trackingNumber`
- `GET /api/trackingdetail/orders`
- `GET /api/trackingdetail/address-change-requests`
- `POST /api/trackingdetail/order/:orderNumber/address-change-request`
- `POST /api/trackingdetail/orders`

### `requireApiKey(req, res, next)`
Express middleware: if `TRACKING_API_KEY` is set, requires a matching `x-api-key` header (401 otherwise); if unset, passes through unconditionally.


### `normalizeEmail(raw)`
Trims/lowercases an email string; returns `''` unless it contains `@` and is ≤150 characters. A cheap sanity filter — the database is the real source of truth for whether the email has any associated data.

**Used by**:
- `requireOtpAccess` (validates the JWT's `email` claim)
- `POST /api/trackingdetail/request-otp`
- `POST /api/trackingdetail/verify-otp`
- `GET /api/trackingdetail/orders` (falls back to `req.query.email` when no OTP token is used)
- `GET /api/trackingdetail/address-change-requests` (same fallback)
- `POST /api/trackingdetail/order/:orderNumber/address-change-request` (same fallback)
- `POST /api/trackingdetail/orders` (same fallback)
- `POST /api/trackingdetail/orders-by-email`

### `generateOtp()`
Returns a random 6-digit numeric string (`100000`–`999999`).

**Used by**: `POST /api/trackingdetail/request-otp` — generates the code emailed to the customer.

### `sha256Hex(raw)`
SHA-256 hashes a string and returns hex. OTPs are never stored in plaintext — only `sha256(salt:otp)`.

**Used by**:
- `POST /api/trackingdetail/request-otp` — hashes the newly generated OTP before storing it.
- `POST /api/trackingdetail/verify-otp` — recomputes the hash from the submitted OTP + stored salt to compare against the stored hash.

---

## External integration

### `fetchTrackingEventsExternal(trackingNumber)`
Calls an external carrier-tracking API (`GET {TRACKING_EVENTS_API_BASE_URL}/track?tracking=...&carrier=...`, bearer-authenticated if `TRACKING_EVENTS_API_KEY` is set), then normalizes whatever shape the provider returns (`{ events: [...] }` or `{ data: { events: [...] } }`) into a flat array of `{ status, location, timestamp, raw }`. Returns `{ ok: false, error }` if `trackingNumber` is missing, the provider isn't configured (`TRACKING_EVENTS_API_BASE_URL` unset), the HTTP call fails, or the response isn't OK — never throws for expected failure modes.

**Used by**: `GET /api/trackingdetail/tracking/:trackingNumber` — the entire handler body.

---

## Core data access

### `fetchTrackingDetailsByEmail(email, limit)`
The main "give me everything about this customer's orders" query. Steps:
1. Normalizes `email` and clamps `limit` to `[1, 200]` (default 50 if not a valid number).
2. `SELECT * FROM orders WHERE LOWER(TRIM(customer_email)) = ? ORDER BY created_at DESC LIMIT <limit>`.
3. If no orders, returns `[]` immediately.
4. Resolves each order's primary key (`id`, falling back to `order_id` if the schema differs) via `getOrderPk`. If none of the orders yield a usable id (unexpected schema), returns the orders with empty `items`/`payments` arrays rather than crashing.
5. Otherwise batch-fetches `order_items` and `payments` for all resolved order IDs in one query each (`WHERE order_id IN (...)`), then groups both by `order_id` into `Map`s.
6. Returns an array of `{ order, items, payments }`, one entry per order, in the original `orders` order.

**Used by**:
- `GET /api/trackingdetail/orders`
- `POST /api/trackingdetail/orders`
- `POST /api/trackingdetail/orders-by-email`

---

## Route handlers (API endpoints)

Base path for all customer-tracking routes: `/api/trackingdetail`. All responses are JSON.

| Method & path | Auth | Purpose | Functions it calls |
|---|---|---|---|
| `GET /health` | None | Liveness probe — runs `SELECT 1` against the DB pool directly (no helper function) and reports `{ ok, service: 'tracking-service', db }`. | — |
| `GET /api/trackingdetail/tracking/:trackingNumber` | `requireOtpAccess` | Looks up carrier tracking events for a tracking number from an external provider. | `requireOtpAccess`, `fetchTrackingEventsExternal` |
| `POST /api/trackingdetail/request-otp` | None (public) | Body `{ email }`. If the email has at least one order, generates a 6-digit OTP, stores its salted hash in `tracking_otps` (expires after `OTP_TTL_SECONDS`), and emails it via `sendEmail(..., 'tracking_otp', ...)`. **Always** responds `{ ok: true }` regardless of whether the email had orders, to avoid leaking which emails exist in the system. | `normalizeEmail`, `generateOtp`, `sha256Hex`, `nowMysqlDatetime`, `addSecondsMysql`, `sendEmail` (from `emailService.js`) |
| `POST /api/trackingdetail/verify-otp` | None (public) | Body `{ email, otp }`. Looks up the most recent OTP row for the email (row-locked with `FOR UPDATE`), and rejects if none exists, already used, expired, or over `OTP_MAX_ATTEMPTS`. On mismatch, increments `attempts` and returns `400`. On match, marks the row used and returns a short-lived JWT (`{ email }`, expires in `ACCESS_TOKEN_TTL_SECONDS`) that the client then presents as a `Bearer` token to the protected routes below. | `normalizeEmail`, `sha256Hex`, `nowMysqlDatetime` |
| `GET /api/trackingdetail/orders` | `requireOtpAccess` | Query param `email` (ignored if the request carries a verified OTP token — `req.otpEmail` wins) and optional `limit` (clamped to 1–200, default 50). Returns `{ email, orders }` where each order includes its items and payments. | `requireOtpAccess`, `normalizeEmail`, `fetchTrackingDetailsByEmail` |
| `POST /api/trackingdetail/orders` | `requireOtpAccess` | Same as the `GET` version but reads `email`/`limit` from the JSON body instead of query params. | `requireOtpAccess`, `normalizeEmail`, `fetchTrackingDetailsByEmail` |
| `POST /api/trackingdetail/orders-by-email` | None (public) | Body `{ email, limit }`. Same underlying lookup as the two routes above, but with no OTP/token requirement at all — intended for callers that have already authenticated the user through some other mechanism. Returns `{ ok: true, orders }`. | `normalizeEmail`, `fetchTrackingDetailsByEmail` |
| `GET /api/trackingdetail/address-change-requests` | `requireOtpAccess` | Query param `email` (or `req.otpEmail`) and optional `limit` (1–500, default 200). Lists that customer's `order_address_change_requests` rows, newest first. | `requireOtpAccess`, `normalizeEmail` |
| `POST /api/trackingdetail/order/:orderNumber/address-change-request` | `requireOtpAccess` | Body `{ shipping_address, shipping_city, shipping_zip, shipping_country }` (only `shipping_address` is required). Locks the target order row, rejects if it doesn't belong to the requesting email or if a `pending` request already exists for it (`409`), snapshots the order's current shipping fields, and inserts a new `pending` row capturing both the current and requested addresses as JSON. | `requireOtpAccess`, `normalizeEmail`, `nowMysqlDatetime` |

### Startup sequence (bottom of file)
```js
app.listen(PORT, ...)                     // starts accepting connections immediately
;(async () => {
  await ensureTrackingOtpsTable()
  await ensureOrderAddressChangeRequestsTable()
})().catch(() => {})
```


---

## External modules referenced (not defined in `index.js`)

| Import | From | Role |
|---|---|---|
| `mysql` (aliased) | `./db-adapter.js` | Drop-in `mysql2/promise`-compatible pool that transparently translates MySQL-dialect SQL/DDL to PostgreSQL and runs it via `pg` against the Supabase database. See `documentation/` (or the file's own header comment) for translation details. |
| `sendEmail` | `../emailService.js` (shared, at `Dev/emailService.js`) | Sends the `tracking_otp` templated email through Resend (primary) or Mailjet (fallback). Same shared module `admin-service` uses its own copy of. |
